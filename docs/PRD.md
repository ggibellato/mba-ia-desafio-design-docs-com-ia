# PRD — Sistema de Webhooks de Notificação de Pedidos

Documento de consolidação. Não redefine nenhuma decisão já registrada em `docs/adrs/`, `docs/RFC.md` ou `docs/FDD.md` — apenas organiza, em linguagem de produto, o que já foi decidido e por quê. Em caso de dúvida sobre um requisito, a fonte é sempre rastreável via `docs/TRACKER.md`.

## Resumo e Contexto da Feature

O Order Management System (OMS) atual não tem nenhum mecanismo de notificação externa: clientes que precisam saber quando o status de um pedido muda fazem polling manual em `GET /orders` (`ANALIZE_CODEBASE.md`). Esta feature adiciona um sistema de webhooks *outbound*: quando o status de um pedido muda, a plataforma notifica automaticamente os endpoints HTTP cadastrados pelo cliente, com garantia de entrega, autenticação e histórico auditável.

## Problema e Motivação

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente notificação em tempo real de mudanças de status de pedido. Hoje eles fazem polling periódico em `GET /orders`, o que a Atlas já descreveu como uma integração lenta e cara; a Atlas sinalizou que pode migrar para um concorrente se a feature não for entregue até o fim do trimestre (`[09:00]`, Marcos). Para os três clientes, "tempo real" significa qualquer latência de notificação abaixo de 10 segundos — não é um requisito de streaming, é eliminar a necessidade de ficar consultando a API manualmente (`[09:02]`, Marcos).

## Público-Alvo e Cenários de Uso

### Público-alvo

- **Clientes B2B integradores** — hoje, especificamente Atlas Comercial, MaxDistribuição e Nova Cargo — que consomem a API do OMS para acompanhar o ciclo de vida dos próprios pedidos.
- **Operadores internos autenticados** — cadastram/gerenciam webhooks em nome de um cliente via API (o cadastro não é feito pelo cliente diretamente logado; é feito por um usuário do nosso sistema que representa o cliente, `[09:31–09:32]`).
- **Administradores internos (role `ADMIN`)** — reprocessam manualmente eventos que falharam definitivamente (DLQ).

### Cenários de uso

- Um operador cadastra um webhook para a Atlas, informando a URL do endpoint dela e os status que ela quer acompanhar (ex.: `SHIPPED`, `DELIVERED`); a plataforma gera e devolve a secret na criação.
- Um pedido da Atlas muda de `PROCESSING` para `SHIPPED`; o sistema notifica o endpoint cadastrado em poucos segundos, sem que a Atlas precise consultar `GET /orders`.
- O endpoint da MaxDistribuição fica temporariamente fora do ar; a plataforma reentrega automaticamente com backoff, sem intervenção manual, até o endpoint voltar.
- O endpoint da Nova Cargo fica indisponível por mais de 15 horas; o evento cai em uma fila de falhas definitivas; depois que a Nova Cargo conserta a integração dela, um administrador reprocessa manualmente o evento.
- Um cliente quer depurar um problema de integração do lado dele e consulta o histórico das últimas entregas de um webhook (sucesso/falha, payload, resposta, tempo de resposta).
- Um cliente rotaciona a secret do webhook por política interna de segurança, sem que isso derrube a validação das notificações em trânsito durante a transição.

## Objetivos e Métricas de Sucesso

- **Objetivo 1 — Eliminar a dependência de polling para acompanhamento de status.**
  Métrica: p95 da latência entre a mudança de status do pedido e a primeira tentativa de entrega do webhook.
  Meta: ≤ 10 segundos (definição de "tempo real" dos clientes, `[09:02]`), medida nas primeiras 4 semanas após o lançamento com os 3 clientes-piloto (Atlas, MaxDistribuição, Nova Cargo). **Meta proposta** — a reunião definiu o limite de aceitação (<10s), mas não definiu explicitamente uma métrica de acompanhamento pós-lançamento; a métrica acima é uma proposta alinhada a esse limite.
- **Objetivo 2 — Reter o cliente Atlas Comercial.**
  Métrica: entrega da feature em produção para os 3 clientes-piloto.
  Meta: até o fim do trimestre corrente (`[09:00]`, prazo citado pela Atlas como condição para não migrar de fornecedor). Estimativa interna da equipe: 3 sprints, incluindo revisão de segurança dedicada (`[09:45–09:47]`).

## Escopo

### Em escopo

- Cadastro, edição, remoção e listagem de webhooks por cliente, incluindo filtro de quais status de pedido cada webhook quer receber (`[09:31–09:33]`).
- Geração e rotação de secret por webhook, com grace period de 24h (`[09:21]`, ADR-004).
- Publicação do evento de notificação de forma atômica com a mudança de status do pedido (ADR-001).
- Entrega assíncrona com retry, backoff e Dead Letter Queue (DLQ) para falhas persistentes (ADR-002, ADR-003).
- Autenticação das notificações via assinatura HMAC-SHA256 (ADR-004).
- Garantia de entrega at-least-once, com identificador único por evento para deduplicação do lado do cliente (ADR-005).
- Histórico consultável das últimas entregas de cada webhook (`[09:34]`).
- Endpoint administrativo para reprocessamento manual de eventos na DLQ, restrito a operadores `ADMIN` (`[09:18–09:19]`, `[09:35–09:36]`).

### Fora de escopo

- **Rate limiting de envio ao cliente** — a equipe decidiu não implementar agora; vai observar o comportamento em produção antes de decidir (`[09:38–09:39]`).
- **Notificação ao cliente (ex.: e-mail) quando o webhook dele falha repetidamente** — adiada para uma fase futura, condicionada a medir o impacto real primeiro (`[09:37–09:38]`).
- **Dashboard/painel visual para o cliente acompanhar os próprios webhooks** — fora de escopo desta feature; considerado projeto separado do time de frontend (`[09:39–09:40]`).
- **Arquivamento de eventos já entregues na tabela de outbox** — mecanismo de limpeza/retenção de longo prazo, fora do escopo desta feature (`[09:08]`).
- **Escalar o worker para múltiplos processos em paralelo** — a garantia de ordering atual depende de um único worker; particionamento fica como problema futuro (ADR-007).

## Requisitos Funcionais

1. **[FR-01]** O sistema deve notificar automaticamente os endpoints de webhook cadastrados quando o status de um pedido mudar, eliminando a necessidade de polling manual em `GET /orders` (`[09:00–09:02]`).
2. **[FR-02]** O cliente (via operador autenticado) deve poder cadastrar um webhook informando `url` e a lista de status de pedido desejados; a secret é gerada pela plataforma e devolvida apenas na criação (`[09:31]`).
3. **[FR-03]** O cliente deve poder editar, remover e listar os webhooks já cadastrados para um customer (`[09:33]`).
4. **[FR-04]** O cliente deve poder escolher, por webhook, quais status de pedido deseja receber — se nenhum webhook do cliente quiser um determinado status, nenhuma notificação é gerada para aquele evento (`[09:33–09:34]`).
5. **[FR-05]** O cliente deve poder consultar o histórico das entregas mais recentes de um webhook, incluindo sucesso/falha, payload, resposta recebida e tempo de resposta (`[09:34]`).
6. **[FR-06]** O sistema deve garantir que a URL cadastrada seja `https`; URLs não seguras devem ser rejeitadas no cadastro (`[09:23]`).
7. **[FR-07]** O cliente deve poder solicitar a rotação da secret de um webhook via API, sem interromper a validação de notificações já em trânsito (`[09:21]`).
8. **[FR-08]** O sistema deve assinar cada notificação enviada com HMAC-SHA256, permitindo ao cliente validar que a notificação veio da plataforma e não foi adulterada (`[09:19–09:20]`).
9. **[FR-09]** O sistema deve reentregar automaticamente uma notificação que falhou, com backoff crescente, antes de considerá-la definitivamente falha (`[09:14–09:17]`).
10. **[FR-10]** O sistema deve manter, de forma consultável, os eventos que esgotaram todas as tentativas de entrega, e deve permitir que um administrador reprocesse manualmente esses eventos (`[09:18–09:19]`, `[09:35–09:36]`).
11. **[FR-11]** O sistema deve garantir que cada notificação seja entregue pelo menos uma vez (at-least-once) e deve incluir um identificador único por evento para que o cliente possa detectar e descartar duplicatas do lado dele (`[09:24–09:26]`).

Onze requisitos funcionais identificados na reunião — acima do mínimo de 8 exigido pelo desafio; não foram adicionados requisitos além do que está diretamente sustentado pela transcrição.

## Requisitos Não Funcionais

- **[NFR-01] Latência de entrega**: p95 de até 10 segundos entre a mudança de status e a primeira tentativa de entrega, refletindo a definição de "tempo real" dos clientes (`[09:02]`); no pior caso operacional (2s de polling + até 10s de timeout de chamada), a primeira tentativa pode levar até ~12s (ADR-002, `[09:42]`).
- **[NFR-02] Confiabilidade de entrega**: nenhum evento é perdido silenciosamente — todo evento que esgota as tentativas de retry fica em um estado observável e reprocessável (DLQ), nunca é simplesmente descartado (ADR-003).
- **[NFR-03] Segurança de transporte e autenticação**: apenas URLs `https` são aceitas; toda notificação é assinada com HMAC-SHA256; cada webhook tem secret própria, não compartilhada entre clientes (`[09:21–09:23]`).
- **[NFR-04] Continuidade na rotação de secret**: a secret anterior permanece válida por 24h após uma rotação, para o cliente migrar sem downtime de validação (`[09:21]`).
- **[NFR-05] Tamanho de payload**: notificações são limitadas a 64KB; eventos que ultrapassarem esse limite não são enviados truncados — geram erro explícito (`[09:23–09:24]`).
- **[NFR-06] Auditabilidade de operações administrativas**: todo reprocessamento manual de evento na DLQ deve ficar registrado, com identificação de quem executou a ação (`[09:35–09:36]`).
- **[NFR-07] Compatibilidade**: a feature é aditiva — nenhum contrato de API já existente (`orders`, `products`, `customers`, `users`, `auth`) é alterado (`docs/FDD.md`, seção "Dependências e Compatibilidade").

## Decisões e Trade-offs Principais

- **Outbox no MySQL, não fila externa** — a inserção do evento acontece na mesma transação da mudança de status, garantindo que os dois nunca fiquem dessincronizados; em troca, a entrega deixa de ser síncrona (ver ADR-001, RFC "Proposta Técnica").
- **Worker separado em polling de 2s, não trigger de banco** — simplicidade operacional em troca de uma latência mínima de 2s no pior caso (ADR-002).
- **At-least-once, não exactly-once** — evita a complexidade de coordenação transacional entre plataforma e cliente; em troca, o cliente precisa deduplicar por conta própria via identificador único do evento (ADR-005).
- **5 tentativas com backoff de até 12h, DLQ com replay manual** — tolera indisponibilidades reais do cliente (já observadas de até 2h) sem manter retry indefinido; em troca, uma falha só é definitivamente sinalizada após quase 15h (ADR-003).
- **Ordering garantido apenas por pedido, apenas com um único worker** — solução simples que atende à necessidade real dos clientes (acompanhar cada pedido individualmente), mas não escala para múltiplos workers sem revisão futura (ADR-007).
- **Reuso máximo dos padrões já existentes no projeto** (erros, logger, autenticação, paginação, estrutura de módulos) em vez de introduzir convenções novas para este módulo — reduz superfície de código novo e mantém consistência com o resto da base (ADR-006).

Detalhamento completo de cada decisão, alternativas descartadas e consequências: `docs/adrs/` e `docs/RFC.md`.

## Dependências

- **MySQL/Prisma já operados pela equipe** — quatro tabelas novas, sem alteração em tabelas existentes (`docs/FDD.md`).
- **Novo processo worker (`src/worker.ts`)** — precisa ser implantado e monitorado como uma peça de infraestrutura adicional, independente da API (ADR-002). *Dependência bloqueante* para qualquer entrega de notificação funcionar.
- **Revisão de segurança dedicada** — Sofia (Engenharia de Segurança) pediu pelo menos 2 dias úteis de revisão de HMAC e geração/rotação de secret antes do deploy em produção (`[09:46]`). *Dependência bloqueante* para o lançamento.
- **Nenhuma dependência de biblioteca externa nova** — assinatura HMAC usa `node:crypto` (nativo) e chamadas HTTP usam `fetch` nativo do Node 20; não bloqueante.
- **Comunicação com os clientes-piloto** — Marcos se comprometeu a documentar o comportamento de deduplicação (`X-Event-Id`) e o processo de integração no portal de desenvolvedor (`[09:26]`, `[09:40]`). *Não bloqueante* para o lançamento técnico, mas necessário para adoção correta pelos clientes.

## Riscos e Mitigação

- **Risco 1 — Vazamento de secret de um cliente.**
  Probabilidade: Média (já aconteceu antes — um cliente vazou uma secret em log de aplicação dele, `[09:22]`).
  Impacto: Alto para o cliente afetado (permitiria forjar notificações em nome da plataforma para aquele endpoint).
  Mitigação: secret exclusiva por endpoint (não compartilhada entre clientes) e rotação com grace period de 24h, limitando o raio de impacto de qualquer vazamento a um único cliente (ADR-004).

- **Risco 2 — Cliente não implementa corretamente a verificação HMAC ou a deduplicação por evento.**
  Probabilidade: Média (depende de uma implementação correta do lado de terceiros).
  Impacto: Médio (cliente pode processar notificações duplicadas ou aceitar notificações não verificadas).
  Mitigação: documentação destacada no portal de desenvolvedor sobre o comportamento at-least-once e o mecanismo de deduplicação (`[09:26]`, `[09:40]`).

- **Risco 3 — Gargalo de throughput por depender de um único worker.**
  Probabilidade: Baixa no volume atual, mas sem caminho de escala pronto se o volume crescer.
  Impacto: Médio (atraso na entrega de notificações sob alto volume).
  Mitigação: particionamento por pedido ou lock pessimista fica identificado como caminho de evolução, mas deliberadamente fora do escopo desta entrega (ADR-007).

- **Risco 4 — Crescimento não controlado da tabela de eventos.**
  Probabilidade: Alta ao longo do tempo, por design (arquivamento está fora de escopo, `[09:08]`).
  Impacto: Baixo no curto prazo, potencialmente médio em performance de consulta no longo prazo.
  Mitigação: índices já previstos em `status`/`created_at`; arquivamento de eventos entregues fica como trabalho futuro reconhecido, não resolvido nesta entrega.

- **Risco 5 — Falha de um cliente ficar sem detecção por até ~15 horas.**
  Probabilidade: Baixa (só ocorre após esgotar as 5 tentativas de retry).
  Impacto: Médio (cliente fica sem receber notificações por um período prolongado antes de qualquer ação corretiva).
  Mitigação: aceito conscientemente como trade-off — o objetivo é não descartar eventos legítimos durante indisponibilidades temporárias mais longas; reprocessamento manual via DLQ está disponível assim que o problema do cliente for resolvido (ADR-003).

## Critérios de Aceitação

- Um webhook cadastrado com sucesso recebe uma secret na resposta de criação, e essa secret nunca mais aparece em texto plano em nenhuma outra resposta da API.
- Uma mudança de status para um valor coberto por um webhook ativo do cliente resulta em uma tentativa de notificação dentro do intervalo de latência esperado (NFR-01).
- Uma notificação entregue com sucesso é assinada com HMAC-SHA256 verificável pela secret do cliente.
- Uma falha de entrega é reentregue automaticamente segundo o cronograma de backoff, sem intervenção manual, até esgotar as tentativas.
- Um evento que esgota todas as tentativas aparece de forma consultável (não é descartado) e pode ser reprocessado manualmente apenas por um usuário com role `ADMIN`.
- Uma tentativa de reprocessar um evento já reprocessado, ou por um usuário sem role `ADMIN`, é rejeitada.
- O histórico de entregas de um webhook reflete corretamente sucesso/falha, payload, resposta e tempo de resposta de cada tentativa.
- Nenhum requisito, decisão ou restrição deste documento contradiz `TRANSCRICAO.md`, os ADRs ou `docs/RFC.md`/`docs/FDD.md`.

## Estratégia de Testes e Validação

- **Testes automatizados**: suíte de testes de integração seguindo o padrão já usado em `tests/orders.test.ts` (Vitest + Supertest + `tests/helpers/factories.ts`), cobrindo o CRUD de webhooks, publicação de evento na outbox a partir de uma mudança de status real, e os cenários de sucesso/falha/DLQ/replay descritos nos Critérios de Aceitação (ver "Critérios de Aceite Técnicos" em `docs/FDD.md`).
- **Validação de segurança**: revisão dedicada de Sofia sobre a implementação de HMAC e o fluxo de geração/rotação de secret antes do deploy em produção (`[09:46]`), conforme já registrado como dependência bloqueante.
- **Validação com clientes-piloto**: lançamento inicial restrito aos três clientes que solicitaram a feature (Atlas Comercial, MaxDistribuição, Nova Cargo), permitindo validar o comportamento em produção antes de uma adoção mais ampla.
- **Validação pós-lançamento das métricas de sucesso**: acompanhamento, nas primeiras 4 semanas, das métricas de observabilidade já especificadas em `docs/FDD.md` (`webhook_delivery_duration_ms` para o Objetivo 1 de latência; `webhook_dead_letter_total` e `webhook_deliveries_total` para taxa de sucesso de entrega), confirmando na prática se a meta de latência p95 ≤ 10s está sendo atingida com os clientes-piloto.
