# Tracker de Rastreabilidade

Mapeia cada item registrado nos documentos do pacote (ADRs, RFC e, à medida que forem escritos, FDD e PRD) à sua origem na transcrição (`TRANSCRICAO.md`) ou no código (`src/`, `prisma/`).

> **Nota de cobertura (2026-07-19):** `docs/PRD.md` e `docs/FDD.md` ainda são placeholders (Fases 4 e 5 do `PLANO_DE_TRABALHO.md`). Este tracker cobre 100% dos itens identificáveis atualmente existentes — os 7 ADRs (`docs/adrs/`) e o RFC (`docs/RFC.md`) — e será estendido com novas linhas assim que o FDD e o PRD forem produzidos, sem remover as linhas já existentes.

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
