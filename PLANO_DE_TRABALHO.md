# Plano de Trabalho — Desafio "Da Reunião ao Documento"

> Guia passo a passo pessoal para completar o desafio. Não é um entregável do exercício (não está listado na estrutura obrigatória) — é um roteiro de execução. Pode ser apagado ou movido para fora do repo antes da entrega final, se preferir.

Referência completa dos requisitos: [`EXERCICIO.md`](./EXERCICIO.md). Este plano só resume/ordena as tarefas; em caso de dúvida, o `EXERCICIO.md` é a fonte da verdade.

## Legenda de status
- [ ] a fazer
- [~] em andamento
- [x] concluído

---

## Fase 0 — Setup [x] concluído

- [x] Fork do repositório base no GitHub (se ainda não feito) e confirmar que este diretório local aponta pro fork. (`origin` = `ggibellato/mba-ia-desafio-design-docs-com-ia`)
- [x] Confirmar que `TRANSCRICAO.md` e `src/`, `prisma/`, `tests/` estão intactos (não serão alterados durante o desafio). (sem alterações locais nesses caminhos)
- [x] Ler `TRANSCRICAO.md` uma vez, na íntegra, sem IA — só para ter contexto próprio antes de delegar leitura à IA.

## Fase 1 — Contextualização com IA

- [x] Pedir à IA uma exploração do código (`src/`, `prisma/schema.prisma`) para mapear: módulos existentes (auth, users, customers, products, orders), máquina de estados do pedido, controle de estoque, auditoria de mudanças de status, padrões de erro/exceções usados.
- [x] Pedir à IA um resumo estruturado da transcrição, separando explicitamente:
  - decisões fechadas
  - requisitos funcionais explícitos
  - restrições
  - pontos de integração com código existente
  - itens descartados
  - itens adiados para fases futuras
  - detalhes técnicos secundários
- [x] Revisar esse resumo manualmente contra a transcrição — é a base de tudo que vem depois. Erros aqui se propagam para todos os documentos.

## Fase 2 — ADRs primeiro (esqueleto das decisões)

As 6 decisões principais que a reunião discute (cobrir pelo menos 5 delas):

- [x] ADR — Padrão Outbox no MySQL (`ADR-001-outbox-no-mysql.md`)
- [x] ADR — Política de retry com backoff e DLQ (`ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md`)
- [x] ADR — Autenticação HMAC-SHA256 com secret por endpoint (`ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md`)
- [x] ADR — Garantia at-least-once com `X-Event-Id` (`ADR-005-garantia-at-least-once-com-x-event-id.md`)
- [x] ADR — Worker em processo separado em polling (`ADR-002-worker-em-processo-separado-com-polling.md`)
- [x] ADR — Reuso dos padrões existentes do projeto (`ADR-006-reuso-dos-padroes-existentes-do-projeto.md`)

Para cada ADR criado em `docs/adrs/ADR-NNN-titulo-em-kebab-case.md`:
- [x] Seção Status
- [x] Seção Contexto (rastreável à transcrição)
- [x] Seção Decisão
- [x] Seção Alternativas Consideradas (≥1 alternativa real ou plausível)
- [x] Seção Consequências (positivas e negativas, trade-off explícito)

Checklist do conjunto:
- [x] Entre 5 e 8 arquivos ADR no total. (7 arquivos: os 6 acima + `ADR-007-ordering-por-order-id-em-topologia-single-worker.md`, cobrindo a garantia de ordering discutida na reunião)
- [x] Pelo menos 1 ADR referencia arquivos/módulos/classes reais do código (`order.service.ts`, `server.ts`, `app-error.ts`, `error.middleware.ts`, `auth.middleware.ts` — referenciados em ADR-001, 002, 003 e 006, todos confirmados existentes).
- [x] Decisões técnicas secundárias (formato de payload, timeouts, headers) — decidido: ficam só no FDD, não viram ADR extra (TLS obrigatório e limite de 64KB foram classificados por Larissa na própria reunião `[09:24]` como requisito não funcional, não decisão arquitetural).

## Fase 3 — RFC (`docs/RFC.md`)

- [ ] Metadados: autor, status, data, revisores (usar os 5 participantes da reunião como revisores).
- [ ] Resumo executivo (TL;DR).
- [ ] Contexto e problema.
- [ ] Proposta técnica (visão geral, sem repetir nível de detalhe do FDD).
- [ ] Alternativas consideradas: **≥2** alternativas reais descartadas na reunião, cada uma com o trade-off que motivou o descarte.
- [ ] Questões em aberto: **≥2** pontos levantados e não decididos/adiados na reunião.
- [ ] Impacto e riscos.
- [ ] Decisões relacionadas: links para os ADRs já escritos na Fase 2.
- [ ] Conferir tamanho: 2 a 4 páginas — se estiver mais longo, provavelmente há detalhe demais de implementação (isso pertence ao FDD).

## Fase 4 — FDD (`docs/FDD.md`)

- [ ] Contexto e motivação técnica.
- [ ] Objetivos técnicos.
- [ ] Escopo e exclusões.
- [ ] Fluxos detalhados: criação do evento na outbox, processamento pelo worker, retry, DLQ.
- [ ] Contratos públicos: **≥4** endpoints HTTP, cada um com payload de exemplo (request + response), headers, status codes, semântica.
- [ ] Matriz de erros previstos com códigos `WEBHOOK_*`.
- [ ] Estratégias de resiliência (timeouts, retries, backoff, fallback).
- [ ] Observabilidade: métricas, logs e tracing (as 3 têm que aparecer).
- [ ] Dependências e compatibilidade.
- [ ] Critérios de aceite técnicos.
- [ ] Riscos e mitigação.
- [ ] **Seção obrigatória "Integração com o sistema existente"**: nomear ≥4 caminhos de arquivo reais do código base e descrever a integração de cada um (ex: como `changeStatus` é estendido, como classes de erro existentes são reaproveitadas).

## Fase 5 — PRD (`docs/PRD.md`)

Produzido por último entre os grandes docs — deve ser consolidação do que já foi decidido em ADRs/RFC/FDD.

- [ ] Resumo e contexto da feature.
- [ ] Problema e motivação.
- [ ] Público-alvo e cenários de uso.
- [ ] Objetivos e métricas de sucesso — **≥1 objetivo com métrica e meta quantitativa**.
- [ ] Escopo (incluso e fora de escopo) — seção "Fora de escopo" com **≥2 itens** explicitamente descartados/adiados na reunião.
- [ ] Requisitos funcionais — **≥8 requisitos** discutidos na reunião.
- [ ] Requisitos não funcionais.
- [ ] Decisões e trade-offs principais.
- [ ] Dependências.
- [ ] Riscos e mitigação — **≥2 riscos**, cada um com probabilidade, impacto e mitigação.
- [ ] Critérios de aceitação.
- [ ] Estratégia de testes e validação.

## Fase 6 — Tracker (`docs/TRACKER.md`)

- [ ] Montar a tabela no formato obrigatório: `ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização`.
- [ ] Varrer PRD, RFC, FDD e cada ADR extraindo os itens rastreáveis.
- [ ] Cobertura ≥80% dos itens identificáveis nos documentos.
- [ ] ≥70% das linhas com Fonte = `TRANSCRICAO` e timestamp válido no formato `[hh:mm] Nome`.
- [ ] ≥5 linhas com Fonte = `CODIGO` e caminho de arquivo real.
- [ ] Regra prática: se não der pra preencher "Localização" com uma origem concreta, o item provavelmente foi inventado — ajustar ou remover do documento de origem.

## Fase 7 — README do processo (`README.md`)

Por último, quando o processo estiver completo.

- [ ] **Sobre o desafio**: 1–2 parágrafos com suas palavras.
- [ ] **Ferramentas de IA utilizadas**: lista com papel de cada uma.
- [ ] **Workflow adotado**: ordem de produção dos documentos, como organizou a interação com a IA.
- [ ] **Prompts customizados**: ≥2 prompts relevantes, em blocos de código.
- [ ] **Iterações e ajustes**: ≥2 momentos concretos em que a IA errou/foi superficial e você corrigiu; quantas iterações principais até o resultado final.
- [ ] **Como navegar a entrega**: caminho dos arquivos e ordem sugerida de leitura.
- [ ] Opcional: manter link/seção referenciando o enunciado original (`EXERCICIO.md`).

## Fase 8 — Revisão final contra os critérios de aceite

Passar item a item pela checklist de "Critérios de Aceite" do `EXERCICIO.md` (seção completa) antes do push final. Pontos de atenção especial:

- [ ] Nenhum requisito/decisão/restrição nos documentos contradiz a transcrição ou o código.
- [ ] Nenhum arquivo de código citado nos documentos é inexistente no repositório real (conferir cada caminho citado com `ls`/busca).
- [ ] Itens explicitamente descartados na reunião NÃO aparecem como requisito em nenhum documento.
- [ ] Estrutura de pastas confere com a estrutura obrigatória do entregável.

## Fase 9 — Push final

- [ ] Commit e push para o fork público no GitHub.
- [ ] Conferir que o repositório está público.

---

## Log de Atividades

Registro cronológico das interações com a IA durante o desafio: quando, qual skill/prompt foi usado, e o que foi produzido. Serve de matéria-prima para as seções "Prompts customizados" e "Iterações e ajustes" do README (Fase 7).

| Data/Hora | Skill/Prompt | Arguments |
|---|---|---|
| 2026-07-18 12:22 | `/analyze-codebase` | Pedir à IA uma exploração do código (`src/`, `prisma/schema.prisma`) para mapear: módulos existentes (auth, users, customers, products, orders), máquina de estados do pedido, controle de estoque, auditoria de mudanças de status, padrões de erro/exceções usados. |
| 2026-07-18 12:43 | `/summarize-meeting-transcript` | resumo estruturado da transcrição, separando explicitamente: decisões fechadas, requisitos funcionais explícitos, restrições, pontos de integração com código existente, itens descartados, itens adiados para fases futuras, detalhes técnicos secundários. |
| 2026-07-19 08:29 | `/write-adrs` | Escrever os ADRs exigidos pelo desafio, usando `EXERCICIO.md`, `RESUMO_TRANSCRICAO.md`, `TRANSCRICAO.md` e `ANALIZE_CODEBASE.md` em ordem de prioridade, sem inventar decisões, alternativas ou consequências não suportadas pelas fontes. |

## Notas de processo

- É esperado 3 a 5 ciclos de geração → revisão crítica → ajuste de prompt → nova geração por documento. Gerar tudo de primeira sem ajustes é sinal de documento genérico demais.
- Prompts vagos ("gere um PRD a partir dessa transcrição") tendem a produzir documentos vazios. Usar prompts dirigidos, citando seções específicas da transcrição/código.
- O tracker é o principal mecanismo de defesa contra alucinação — pode valer a pena começar a esboçar linhas do tracker à medida que cada documento é escrito, em vez de deixar tudo pro fim.
