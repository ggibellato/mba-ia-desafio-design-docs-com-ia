# Tracker de Rastreabilidade

Mapeia cada item registrado nos documentos do pacote (ADRs, RFC, FDD e PRD) à sua origem na transcrição (`TRANSCRICAO.md`) ou no código (`src/`, `prisma/`).

> **Nota de cobertura (2026-07-19):** Pacote de documentos completo — os 7 ADRs (`docs/adrs/`), o RFC (`docs/RFC.md`), o FDD (`docs/FDD.md`) e o PRD (`docs/PRD.md`). Este tracker cobre 100% dos itens identificáveis nesses documentos.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Evento de webhook inserido em `webhook_outbox` na mesma transação que atualiza `order` e `order_status_history` | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Fila externa dedicada (Redis Streams) descartada por overengineering para um time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Entrega síncrona dentro de `OrderService.changeStatus` descartada por risco de travar outros pedidos e impossibilidade de rollback | TRANSCRICAO | [09:03–09:04] Bruno |
| ADR-001-TO-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Entrega deixa de ser síncrona; existe uma janela de tempo entre a mudança de status e o envio do webhook | TRANSCRICAO | [09:04] Bruno |
| ADR-001-COD-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Integração com Código | `OrderService.changeStatus` já executa dentro de `prisma.$transaction`, ponto de inserção do evento na outbox | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker roda como processo Node separado (`src/worker.ts`), com polling de 2 segundos na outbox | TRANSCRICAO | [09:10] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Alternativa Descartada | Trigger de banco de dados para notificação reativa, descartado por MySQL não ter listener nativo | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Alternativa Descartada | Worker rodando dentro do mesmo processo da API, descartado por acoplar a disponibilidade do worker à da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-TO-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Trade-off | Latência mínima de entrega de 2 segundos no pior caso, aceita pela equipe | TRANSCRICAO | [09:10] Larissa |
| ADR-002-COD-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Integração com Código | `src/server.ts` é o padrão de entry-point existente replicado pelo novo `src/worker.ts` | CODIGO | src/server.ts |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Decisão | Retry com backoff exponencial (5 tentativas, 1m/5m/30m/2h/12h) e Dead Letter Queue com replay manual via endpoint admin | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Alternativa Descartada | 3 tentativas de retry, descartada por insuficiente diante de indisponibilidades reais de até 2h | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Alternativa Descartada | Retry indefinido sem teto, descartado por deixar eventos pendurados indefinidamente | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Alternativa Descartada | Marcar falha permanente na própria outbox (`status = failed`), sem tabela DLQ dedicada | TRANSCRICAO | [09:18] Diego |
| ADR-003-TO-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Trade-off | Total de ~15 horas entre a primeira falha e a falha permanente, considerado aceitável pela equipe | TRANSCRICAO | [09:17] Diego |
| ADR-003-COD-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Integração com Código | `requireRole` reaproveitado para restringir o endpoint de replay da DLQ à role ADMIN | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | Autenticação HMAC-SHA256 sobre o corpo da requisição, secret exclusiva por endpoint, rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Secret global única para toda a plataforma, rejeitada por ampliar o raio de impacto de um vazamento | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Rotação de secret sem grace period, descartada por quebrar a integração do cliente no momento da troca | TRANSCRICAO | [09:21] Sofia |
| ADR-004-TO-01 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Durante o grace period de 24h, duas secrets ficam simultaneamente válidas para o mesmo endpoint | TRANSCRICAO | [09:21] Sofia |
| ADR-005 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | Garantia de entrega at-least-once, com deduplicação client-side via header `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT-01 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Alternativa Descartada | Garantia exactly-once, descartada por exigir coordenação transacional complexa entre plataforma e cliente | TRANSCRICAO | [09:25] Diego |
| ADR-005-TO-01 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de deduplicar eventos duplicados é transferida para o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Módulo de webhooks replica a estrutura controller/service/repository/routes/schemas já usada nos demais módulos, reaproveitando `AppError`, Pino, error middleware e `requireRole` | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Alternativa Descartada | Criar hierarquia de erros e logger próprios para o módulo de webhooks, isolados do resto do projeto | TRANSCRICAO | [09:28–09:29] Bruno |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração com Código | Classe base `AppError`, estendida pelas novas exceções `WEBHOOK_*` | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-COD-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração com Código | Subclasses de erro HTTP existentes (`InsufficientStockError`, `InvalidStatusTransitionError`) usadas como modelo para os erros de webhook | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-COD-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração com Código | Middleware de erro centralizado já trata `AppError`, `ZodError` e erros do Prisma, sem exigir alteração | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-COD-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração com Código | Logger Pino já usado no projeto inteiro, reaproveitado sem alteração | CODIGO | src/shared/logger/index.ts |
| ADR-006-COD-05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Integração com Código | Padrão de validação de entrada com schemas Zod, já usado em todos os módulos existentes | CODIGO | src/modules/orders/order.schemas.ts |
| ADR-007 | docs/adrs/ADR-007-ordering-por-order-id-em-topologia-single-worker.md | Decisão | Ordering de entrega garantido apenas por `order_id`, apenas em topologia single-worker; sem garantia de ordering global | TRANSCRICAO | [09:13] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-ordering-por-order-id-em-topologia-single-worker.md | Alternativa Descartada | Particionamento por `order_id` ou lock pessimista para múltiplos workers, adiado como problema futuro | TRANSCRICAO | [09:13] Diego |
| ADR-007-COD-01 | docs/adrs/ADR-007-ordering-por-order-id-em-topologia-single-worker.md | Integração com Código | Índices existentes em `orderId` (`OrderItem`, `OrderStatusHistory`) e `createdAt` (`Order`), usados como analogia para a ordenação da outbox | CODIGO | prisma/schema.prisma |
| RFC-CTX-01 | docs/RFC.md | Restrição | "Tempo real", para os clientes, é qualquer latência de notificação abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-02 | docs/RFC.md | Restrição | Risco de a Atlas Comercial migrar para um concorrente se a feature não for entregue até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Entrega síncrona dentro de `OrderService.changeStatus` | TRANSCRICAO | [09:03–09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Fila externa dedicada (Redis Streams) | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Notificação reativa via trigger de banco de dados | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Garantia de entrega exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-Q-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio de webhooks ainda não decidido; equipe vai observar o comportamento em produção antes de decidir | TRANSCRICAO | [09:38–09:39] Diego |
| RFC-Q-02 | docs/RFC.md | Questão em Aberto | Nível de permissão do CRUD de configuração de webhook pode precisar ser endurecido no futuro, sem critério definido | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-03 | docs/RFC.md | Questão em Aberto | Notificação ao cliente sobre falhas recorrentes (ex.: e-mail) ficou fora do escopo desta fase, sem compromisso de prazo | TRANSCRICAO | [09:37–09:38] Larissa |
| RFC-Q-04 | docs/RFC.md | Questão em Aberto | Mecanismo exato de auditoria do replay manual de eventos da DLQ não foi especificado na reunião | TRANSCRICAO | [09:35–09:36] Sofia |
| RFC-RISK-01 | docs/RFC.md | Risco | Vazamento de secret de cliente — já ocorreu antes, em log de aplicação de um cliente | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-02 | docs/RFC.md | Risco | Gargalo de throughput em topologia de worker único, sem caminho de escala pronto | TRANSCRICAO | [09:13] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Crescimento não controlado da tabela `webhook_outbox`; arquivamento de eventos entregues ficou fora do escopo desta feature | TRANSCRICAO | [09:08] Diego |
| RFC-RISK-04 | docs/RFC.md | Risco | Cliente pode não implementar corretamente a verificação HMAC ou a deduplicação por `X-Event-Id` | TRANSCRICAO | [09:25] Sofia |
| RFC-RISK-05 | docs/RFC.md | Risco | Latência de até ~15 horas até uma falha ser definitivamente classificada como permanente | TRANSCRICAO | [09:16–09:17] Diego |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Publicação do evento na outbox dentro da mesma transação de `OrderService.changeStatus` | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker processa eventos pendentes em lote, em polling de 2 segundos | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry com backoff exponencial de 5 tentativas (1m/5m/30m/2h/12h) | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Evento esgotado vai para `WebhookDeadLetter`; reprocessamento manual via endpoint admin | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /api/v1/webhooks` — cadastro de webhook, secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /api/v1/webhooks` — listagem paginada dos webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /api/v1/webhooks/:id` — edição de url/eventos/status ativo | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /api/v1/webhooks/:id` — remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `POST /api/v1/webhooks/:id/secret/rotate` — rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `GET /api/v1/webhooks/:id/deliveries` — histórico das últimas entregas (sucesso/falha, payload, response, tempo de resposta) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay manual de evento da DLQ, role ADMIN | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-01 | docs/FDD.md | Restrição | Código de erro `WEBHOOK_NOT_FOUND` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Restrição | Código de erro `WEBHOOK_INVALID_URL` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Restrição | Código de erro `WEBHOOK_SECRET_REQUIRED` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Restrição | Código de erro `WEBHOOK_PAYLOAD_TOO_LARGE`, payload de entrega limitado a 64KB | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-05 | docs/FDD.md | Restrição | Código de erro `WEBHOOK_DELIVERY_TIMEOUT`, timeout de 10s por chamada HTTP de entrega | TRANSCRICAO | [09:42] Diego |
| FDD-INTEGRACAO-01 | docs/FDD.md | Integração com Código | `OrderService.changeStatus` estendido para chamar `publishWebhookEvent(tx, ...)` | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEGRACAO-02 | docs/FDD.md | Integração com Código | `buildControllers` ganha `WebhookRepository`/`WebhookService`/`WebhookController`, mesmo padrão de `orders` | CODIGO | src/app.ts |
| FDD-INTEGRACAO-03 | docs/FDD.md | Integração com Código | `buildApiRouter` ganha `router.use('/webhooks', ...)` e `router.use('/admin/webhooks', ...)` | CODIGO | src/routes/index.ts |
| FDD-INTEGRACAO-04 | docs/FDD.md | Integração com Código | Novas classes de erro `WEBHOOK_*` estendem `AppError`, no molde de `InsufficientStockError`/`InvalidStatusTransitionError` | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEGRACAO-05 | docs/FDD.md | Integração com Código | `requireRole('ADMIN')` reaproveitado sem alteração no endpoint de replay de DLQ | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEGRACAO-06 | docs/FDD.md | Integração com Código | `paginated()` reaproveitado sem alteração em `GET /webhooks` e `GET /webhooks/:id/deliveries` | CODIGO | src/shared/http/response.ts |
| FDD-INTEGRACAO-07 | docs/FDD.md | Integração com Código | `redactPaths` do logger precisa ganhar entrada para o campo `secret` do webhook | CODIGO | src/shared/logger/index.ts |
| FDD-INTEGRACAO-08 | docs/FDD.md | Integração com Código | Padrão de bootstrap de `src/server.ts` (PrismaClient próprio, shutdown gracioso) replicado em `src/worker.ts` | CODIGO | src/server.ts |
| FDD-INTEGRACAO-09 | docs/FDD.md | Integração com Código | Quatro modelos novos (`Webhook`, `WebhookOutboxEvent`, `WebhookDelivery`, `WebhookDeadLetter`) adicionados sem alterar modelos existentes | CODIGO | prisma/schema.prisma |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Latência p95 de entrega ≤ 10s, refletindo a definição de "tempo real" dos clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | Entrega da feature até o fim do trimestre para reter o cliente Atlas Comercial | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Sistema deve notificar automaticamente clientes na mudança de status, eliminando polling manual | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Cadastro de webhook via POST (url + status desejados), secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Edição, remoção e listagem de webhooks cadastrados | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Filtro de quais status de pedido cada webhook deseja receber | TRANSCRICAO | [09:33–09:34] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Consulta ao histórico das entregas mais recentes de um webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | URL do webhook deve ser https; URLs não seguras são rejeitadas | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Rotação de secret via API, sem interromper validação em trânsito | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 em cada notificação enviada | TRANSCRICAO | [09:19–09:20] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Reentrega automática com backoff antes de considerar falha definitiva | TRANSCRICAO | [09:15] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Eventos esgotados ficam consultáveis (DLQ) e reprocessáveis por ADMIN | TRANSCRICAO | [09:18–09:19] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Garantia at-least-once com identificador único por evento para dedup do cliente | TRANSCRICAO | [09:24–09:26] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega: p95 até 10s, pior caso ~12s | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Nenhum evento perdido silenciosamente — sempre observável e reprocessável | TRANSCRICAO | [09:18] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Apenas URLs https são aceitas para webhooks | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Secret anterior válida por 24h após rotação (grace period) | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Payload de notificação limitado a 64KB, erro explícito se ultrapassar | TRANSCRICAO | [09:23–09:24] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Reprocessamento manual de DLQ deve ficar auditável (quem executou) | TRANSCRICAO | [09:35–09:36] Sofia |
| PRD-ESCOPO-01 | docs/PRD.md | Restrição | Fora de escopo: rate limiting de envio ao cliente | TRANSCRICAO | [09:38–09:39] Diego |
| PRD-ESCOPO-02 | docs/PRD.md | Restrição | Fora de escopo: notificação (ex.: e-mail) sobre falhas recorrentes | TRANSCRICAO | [09:37–09:38] Larissa |
| PRD-ESCOPO-03 | docs/PRD.md | Restrição | Fora de escopo: dashboard/painel visual de webhooks para o cliente | TRANSCRICAO | [09:39–09:40] Larissa |
| PRD-ESCOPO-04 | docs/PRD.md | Restrição | Fora de escopo: arquivamento de eventos já entregues na outbox | TRANSCRICAO | [09:08] Diego |
| PRD-ESCOPO-05 | docs/PRD.md | Restrição | Fora de escopo: escalar o worker para múltiplos processos em paralelo | TRANSCRICAO | [09:13] Diego |
| PRD-RISCO-01 | docs/PRD.md | Risco | Vazamento de secret de cliente (já ocorreu antes) | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-02 | docs/PRD.md | Risco | Cliente não implementa corretamente HMAC ou deduplicação | TRANSCRICAO | [09:25] Sofia |
| PRD-RISCO-03 | docs/PRD.md | Risco | Gargalo de throughput por depender de worker único | TRANSCRICAO | [09:13] Diego |
| PRD-RISCO-04 | docs/PRD.md | Risco | Crescimento não controlado da tabela de eventos | TRANSCRICAO | [09:08] Diego |
| PRD-RISCO-05 | docs/PRD.md | Risco | Falha de cliente sem detecção por até ~15 horas | TRANSCRICAO | [09:16–09:17] Diego |
