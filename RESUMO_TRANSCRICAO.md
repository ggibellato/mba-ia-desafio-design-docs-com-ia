# Resumo Estruturado da Transcrição

Resumo estruturado da reunião técnica "Sistema de Webhooks de Notificação de Pedidos" (`TRANSCRICAO.md`), separado em decisões fechadas, requisitos funcionais explícitos, restrições, pontos de integração com código existente, itens descartados, itens adiados para fases futuras e detalhes técnicos secundários. A transcrição é a fonte de verdade; nenhum conteúdo abaixo foi inferido além do que está registrado nela.

## Decisões fechadas

- Webhooks são exclusivamente *outbound* (saída da plataforma para os clientes); os clientes não enviam webhooks para o sistema. `[09:02–09:03]`
- Entrega assíncrona via padrão Outbox no MySQL — tabela `webhook_outbox`, inserida na mesma transação SQL que já atualiza `orders` e `order_status_history`. `[09:06–09:08]`
- A tabela outbox terá índice em `status` e `created_at`; linhas já entregues são arquivadas após ~30 dias, fora do escopo desta feature. `[09:08]`
- Worker roda como processo separado da API, fazendo *polling* a cada 2 segundos nos eventos pendentes mais antigos. `[09:09–09:11]`
- Worker é um entry-point próprio (ex.: `src/worker.ts`, script `npm run worker`), conectando ao mesmo banco mas com sua própria instância de `PrismaClient` (`PrismaClient` é por processo). `[09:11, 09:29–09:30]`
- Ordering garantido apenas por `order_id` e apenas enquanto houver um único worker; não há garantia de ordering global. Documentado como limitação conhecida. `[09:12–09:14]`
- Retry com backoff exponencial: 5 tentativas, intervalos de 1min / 5min / 30min / 2h / 12h (~15h no total), depois considerado falha permanente. `[09:15–09:17]`
- Falhas permanentes vão para tabela separada `webhook_dead_letter` (payload, motivo da falha, timestamp). `[09:18]`
- Reprocessamento da DLQ é manual, via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento como pendente na outbox. Exige role `ADMIN` (reaproveitando `requireRole` já existente) e deve registrar quem executou o replay, para auditoria. `[09:18–09:19, 09:35–09:36]`
- Autenticidade/integridade dos webhooks via HMAC-SHA256 sobre o corpo da requisição, assinatura enviada no header `X-Signature`. `[09:19–09:20]`
- Secret é única por endpoint de webhook cadastrado — não é uma secret global da plataforma. `[09:21]`
- Secret é rotacionável pela API; a secret antiga permanece válida por 24h em paralelo (grace period) após a rotação. `[09:21]`
- TLS obrigatório: a URL do webhook deve ser `https`; `http` é rejeitado na validação de schema (Zod). `[09:23]`
- Limite de tamanho de payload de 64KB; payload que ultrapasse o limite gera erro (não é truncado). Tratado como requisito não funcional, não como decisão arquitetural separada. `[09:23–09:24]`
- Garantia de entrega *at-least-once* (não *exactly-once*); o cliente pode receber o mesmo evento mais de uma vez. `[09:24–09:26]`
- Deduplicação fica a cargo do cliente, via header `X-Event-Id` (UUID único por evento, gerado na inserção na outbox). `[09:25]`
- Estrutura de código do módulo webhook segue o padrão já existente: `src/modules/webhooks` com controller, service, repository, routes e schemas. `[09:27–09:28]`
- Lógica de processamento do worker fica em arquivo dentro do módulo (`webhook.worker.ts` ou `webhook.processor.ts`); entry-point separado em `src/worker.ts`. `[09:28]`
- Erros seguem o padrão existente (`AppError` e subclasses), com prefixo de código `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`). `[09:28–09:29]`
- Reuso de infraestrutura já existente: logger Pino e middleware de erro centralizado (já trata `AppError`, `ZodError` e erros do Prisma, sem necessidade de alteração). `[09:29]`
- Filtro de eventos por status é aplicado no momento da inserção na outbox, não no momento do envio: se nenhum webhook do customer quiser aquele status, o evento nem é inserido. `[09:33–09:34]`
- Endpoints de CRUD de configuração de webhook (cadastrar, editar, remover, listar) são autenticados normalmente, com qualquer role autenticada — diferente do endpoint de replay de DLQ, que exige `ADMIN`. `[09:36–09:37]`
- `customer_id` é passado no body/path da requisição, não extraído do JWT — o JWT é do usuário operador que representa o cliente, autenticando via API própria (não existe painel do cliente nesta fase). `[09:31–09:32]`
- Payload do evento inclui: `event_id`, `event_type` (`"order.status_changed"`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido (ex.: `total_cents`) — sem os itens do pedido, para manter o payload enxuto. `[09:43]`
- Payload é renderizado ("snapshot") no momento da inserção na outbox, refletindo o estado do pedido naquele instante, e não recalculado no momento do envio. `[09:51–09:52]`
- Headers do request de webhook: `X-Event-Id` (UUID), `X-Signature` (HMAC), `X-Timestamp` (timestamp do envio), `X-Webhook-Id` (id do cadastro de webhook), `Content-Type: application/json`. `[09:44–09:45]`
- Timeout do HTTP call do worker: 10 segundos; se o cliente não responder nesse prazo, é tratado como falha e entra no fluxo de retry. `[09:42]`
- Identificadores da outbox usam UUID, seguindo o padrão do restante do projeto. `[09:51]`
- Inserção do evento na `webhook_outbox` ocorre na mesma transação de `OrderService.changeStatus` (a mesma que atualiza `order`, `order_status_history` e estoque); se a inserção falhar, toda a transação sofre rollback. `[09:40–09:41]`
- Integração implementada via função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o client de transação (`tx`) atual, em vez de o `OrderService` precisar injetar um repository inteiro de webhook. `[09:41]`
- Estimativa de prazo: três sprints, incluindo revisão de segurança da Sofia ao final. Sofia pede reserva de pelo menos 2 dias úteis para essa revisão (HMAC e geração de secret) antes do deploy. `[09:45–09:47]`

## Requisitos funcionais explícitos

- Cliente deve poder cadastrar um webhook via `POST`, informando `url` e a lista de status desejados; a secret é gerada pela plataforma e devolvida na criação. `[09:31]`
- Cliente deve poder editar (`PATCH`), remover (`DELETE`) e listar (`GET`) os webhooks cadastrados de um customer. `[09:33]`
- Cliente deve poder escolher, por endpoint de webhook, quais status de pedido deseja receber (filtro de eventos). `[09:33]`
- Cliente deve poder consultar o histórico de entregas de um webhook — `GET /webhooks/:id/deliveries` — com os últimos 100 envios (sucesso/falha, payload, response, tempo de resposta). `[09:34]`
- Deve existir endpoint administrativo para reprocessar manualmente um evento da DLQ — `POST /admin/webhooks/dead-letter/:id/replay` — restrito à role `ADMIN`. `[09:18–09:19, 09:35]`
- Cliente deve poder solicitar rotação da secret de um webhook via API. `[09:21]`
- Sistema deve validar que a URL do webhook é `https` antes de aceitar o cadastro. `[09:23]`

## Restrições

- "Tempo real", na visão dos clientes, é qualquer latência abaixo de 10 segundos. `[09:02]`
- Entrega não pode ser síncrona dentro da transação de mudança de status: risco de travar mudanças de status de outros pedidos por causa de um cliente lento, e impossibilidade de fazer rollback se o cliente estiver fora do ar. `[09:04]`
- Time pequeno — evitar subir nova infraestrutura (ex.: Redis Cluster) quando o MySQL já existente resolve. `[09:07]`
- MySQL não tem listener nativo (tipo `NOTIFY`/`LISTEN` do Postgres); notificação reativa via trigger de banco não é viável sem soluções improvisadas — motivou a escolha por polling. `[09:09]`
- Worker deve rodar como processo separado da API, para não cair junto em reinícios da API. `[09:11]`
- `PrismaClient` é por processo — o worker precisa de instância própria, mesmo conectando ao mesmo banco/`DATABASE_URL`. `[09:29–09:30]`
- Limite de payload: 64KB. `[09:23–09:24]`
- Timeout do HTTP call: 10 segundos. `[09:42]`
- Prazo solicitado pelo cliente Atlas: fim de novembro / fim do trimestre; estimativa interna de três sprints, incluindo revisão de segurança. `[09:00 contexto Marcos, 09:45–09:46]`
- Reserva de pelo menos 2 dias úteis para revisão de segurança da Sofia antes do deploy. `[09:46]`

## Pontos de integração com código existente

- `OrderService.changeStatus` precisa inserir um evento na tabela `webhook_outbox` dentro da mesma transação que já atualiza `order`, insere em `order_status_history` e decrementa `stockQuantity`. `[09:32, 09:40–09:41]`
- Nova função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`, recebendo o client de transação (`tx`) atual, evitando acoplar o `OrderService` a um repository inteiro de webhook. `[09:41]`
- Módulo webhook segue o padrão estrutural já existente em `src/modules` (controller, service, repository, routes, schemas), replicado em `src/modules/webhooks`. `[09:27]`
- Hierarquia de erros existente (`AppError` e subclasses como `InsufficientStockError`, `InvalidStatusTransitionError`) é reaproveitada; novos erros de webhook usam o prefixo de código `WEBHOOK_`. `[09:28]`
- Logger Pino, já usado no projeto inteiro, é reaproveitado sem alterações. `[09:29]`
- Middleware de erro centralizado (já trata `AppError`, `ZodError` e erros do Prisma) é reaproveitado sem necessidade de mudança. `[09:29]`
- `requireRole`, mecanismo de autorização já existente, é reaproveitado para restringir o endpoint de replay de DLQ à role `ADMIN`. `[09:36]`
- Padrão de schemas Zod do projeto é reaproveitado para validação (ex.: validação de URL `https`). `[09:23, 09:30]`
- Novo entry-point `src/worker.ts`, análogo ao já existente `src/server.ts`. `[09:11]`
- Padrão de identificadores UUID do restante do projeto é replicado na tabela de outbox. `[09:51]`

## Itens descartados

- Entrega síncrona dentro do service de orders — descartada por risco de travar mudanças de status de outros pedidos e por impossibilidade de rollback se o cliente estiver fora do ar. `[09:03–09:04]`
- Uso de Redis Streams (ou fila externa similar) — descartado por representar overengineering para um time pequeno, quando o outbox no MySQL existente já resolve. `[09:07]`
- Uso de trigger de banco de dados para notificar o worker de forma reativa — descartado porque MySQL não tem listener nativo, e as alternativas (escrever em arquivo, chamar um endpoint a partir do trigger) foram consideradas inadequadas. `[09:09]`
- Retry com 3 tentativas — descartado a favor de 5, por ser considerado insuficiente diante de indisponibilidades maiores do cliente (ex.: 2 horas de manutenção planejada já observadas). `[09:16]`
- Retry indefinido com backoff — descartado por deixar eventos "pendurados" indefinidamente se o cliente desaparecer. `[09:15]`
- Truncar o payload que ultrapassar o limite de tamanho — descartado a favor de retornar erro. `[09:23]`
- Garantia de entrega *exactly-once* — descartada por exigir coordenação complexa entre os dois lados; optou-se por *at-least-once* com deduplicação via `X-Event-Id` no lado do cliente. `[09:25]`

## Itens adiados para fases futuras

- Notificação por e-mail ao cliente quando o webhook dele falha repetidamente (ex.: 3 falhas seguidas) — fora de escopo desta fase; possível fase futura, após medir impacto. `[09:37–09:38]`
- Rate limiting de envio de webhooks para o cliente (ex.: muitos pedidos mudando de status no mesmo minuto) — não incluído no escopo atual; item em aberto para observação, decisão adiada. `[09:38–09:39]`
- Dashboard/painel visual para o cliente acompanhar seus webhooks — fora de escopo desta fase; considerado projeto separado do time de frontend. `[09:39–09:40]`
- Endurecimento futuro das permissões do CRUD de configuração de webhook (hoje aberto a qualquer role autenticada) — mencionado como possível ajuste futuro, sem decisão nem prazo definidos. `[09:37]`
- Particionamento por `order_id` ou uso de lock pessimista para permitir múltiplos workers em paralelo mantendo ordering — adiado, tratado como "problema do futuro". `[09:13]`
- Arquivamento de linhas já entregues da outbox após ~30 dias — mencionado como fora do escopo desta feature. `[09:08]`

## Detalhes técnicos secundários

- Tabela `webhook_outbox` terá índice em `status` (pendente, processando, falhou, entregue) e em `created_at`, para permitir leitura eficiente em batch pelos eventos pendentes mais antigos. `[09:08]`
- `customer_id`, `url`, `secret` e estado ativo/inativo compõem a estrutura da tabela de configuração de um webhook cadastrado. `[09:21]`
- Estimativa de esforço por bloco: modelagem de outbox e DLQ (1 sprint), worker e retry (1 sprint), CRUD de configuração e deliveries (0,5 sprint), integração no `order.service` e testes ponta a ponta (0,5 sprint), HMAC/schemas/validações (tempo adicional não quantificado). `[09:45–09:46]`
- Marcos vai documentar o comportamento *at-least-once*/deduplicação e o processo de integração, de forma destacada, no portal de desenvolvedor para os clientes. `[09:26, 09:40]`
- Marcos confirma que vai atualizar os clientes (Atlas, MaxDistribuição, Nova Cargo) sobre o prazo ainda no mesmo dia da reunião. `[09:47]`
- Larissa vai abrir o documento de design da feature e marcar uma sessão de revisão com Bruno e Diego antes do início da implementação. `[09:50]`
- **Ambiguidade não resolvida na transcrição**: ficou definido que o replay de DLQ "deve logar quem fez o replay, pra auditoria" (`[09:36]`, Sofia), mas a transcrição não especifica o mecanismo — se reaproveita alguma tabela de auditoria existente, gera uma nova, ou apenas registra via log Pino. Requer decisão explícita em fase de design/ADR.
