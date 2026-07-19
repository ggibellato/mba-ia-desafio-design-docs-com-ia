# FDD — Sistema de Webhooks de Notificação de Pedidos

Documento de especificação de implementação. Assume como já decididos os 7 ADRs em `docs/adrs/` e a proposta arquitetural em `docs/RFC.md`; este documento não repete o "porquê" dessas decisões, apenas detalha o "como construir". Projeto **não greenfield**: toda integração abaixo referencia arquivos reais do OMS existente.

## Contexto e Motivação Técnica

O `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`) já executa, dentro de uma única transação Prisma, a atualização de `order`, a inserção em `order_status_history` e o débito/reposição de estoque. Esta feature estende essa mesma transação para também inserir eventos de webhook (ADR-001), sem introduzir nenhum componente de infraestrutura novo além do MySQL já operado (ADR-001) e de um segundo processo Node para o worker (ADR-002).

Motivação técnica central: hoje a aplicação não expõe nenhum mecanismo de notificação externa (`ANALIZE_CODEBASE.md`). Os clientes fazem polling em `GET /orders`. Este documento especifica o módulo `src/modules/webhooks` (controller/service/repository/routes/schemas, seguindo o padrão de `src/modules/orders`), o novo entry-point `src/worker.ts` e o modelo de dados necessário para publicar, entregar, reprocessar e auditar esses eventos.

## Objetivos Técnicos

- Publicar um evento de webhook de forma atômica com a mudança de status do pedido (nenhuma mudança de status sem o evento correspondente e vice-versa) — ADR-001.
- Entregar eventos pendentes em até 2 segundos de polling, com timeout de 10s por chamada HTTP — ADR-002, `[09:42]`.
- Garantir at-least-once, nunca zero-vezes, com dedup client-side via `X-Event-Id` — ADR-005.
- Autenticar cada entrega com HMAC-SHA256 e secret por endpoint, com rotação sem downtime (grace period 24h) — ADR-004.
- Não perder eventos definitivamente sem passar por um estado observável e reprocessável (DLQ) — ADR-003.
- Reaproveitar 100% da infraestrutura horizontal já existente (erros, logger, autenticação, validação, paginação) — ADR-006.

## Escopo e Exclusões

### Em escopo

- CRUD de configuração de webhook por cliente, incluindo rotação de secret.
- Publicação de evento na outbox a partir de `OrderService.changeStatus`.
- Worker de entrega com retry, backoff e DLQ.
- Histórico de entregas por webhook.
- Endpoint administrativo de replay manual de DLQ.

### Fora de escopo (decidido na reunião, não implementar)

- Rate limiting de envio ao cliente — adiado, "observar e decidir depois" (`[09:38–09:39]`, RFC-Q-01).
- Notificação ao cliente (ex.: e-mail) sobre falhas recorrentes — adiada para fase futura (`[09:37–09:38]`, RFC-Q-03).
- Dashboard/painel visual de webhooks para o cliente — fora de escopo, projeto separado do frontend (`[09:39–09:40]`).
- Arquivamento de eventos já entregues na outbox — fora do escopo desta feature (`[09:08]`).
- Particionamento do worker para múltiplos processos em paralelo — problema futuro (ADR-007).

## Fluxos Detalhados

### Modelo de dados novo (resumo)

Quatro tabelas novas em `prisma/schema.prisma`, seguindo o padrão de identificadores UUID já usado no restante do schema (`[09:51]`):

```prisma
model Webhook {
  id                     String    @id @default(uuid()) @db.Char(36)
  customerId             String    @db.Char(36)
  url                    String    @db.VarChar(2048)
  secret                 String    @db.VarChar(255)
  previousSecret         String?   @db.VarChar(255)
  previousSecretExpiresAt DateTime?
  events                 Json      // array de OrderStatus, ex.: ["SHIPPED", "DELIVERED"]
  active                 Boolean   @default(true)
  createdAt              DateTime  @default(now())
  updatedAt              DateTime  @updatedAt
}

model WebhookOutboxEvent {
  id            String    @id @default(uuid()) @db.Char(36) // também usado como X-Event-Id
  webhookId     String    @db.Char(36)
  orderId       String    @db.Char(36)
  eventType     String    @db.VarChar(64) // "order.status_changed"
  payload       Json                       // snapshot renderizado na inserção
  status        String    @db.VarChar(16)  // PENDING | PROCESSING | DELIVERED | FAILED
  attempts      Int       @default(0)
  nextAttemptAt DateTime  @default(now())
  createdAt     DateTime  @default(now())

  @@index([status, nextAttemptAt])
  @@index([createdAt])
}

model WebhookDelivery {
  id               String   @id @default(uuid()) @db.Char(36)
  webhookId        String   @db.Char(36)
  outboxEventId    String   @db.Char(36)
  outcome          String   @db.VarChar(16) // SUCCESS | FAILURE
  httpStatus       Int?
  responseSnippet  String?  @db.VarChar(500)
  durationMs       Int
  attemptedAt      DateTime @default(now())

  @@index([webhookId, attemptedAt])
}

model WebhookDeadLetter {
  id            String    @id @default(uuid()) @db.Char(36)
  outboxEventId String    @db.Char(36)
  webhookId     String    @db.Char(36)
  payload       Json
  failureReason String    @db.VarChar(500)
  failedAt      DateTime  @default(now())
  replayedAt    DateTime?
}
```

`WebhookOutboxEvent.status` usa string em vez de enum Prisma para evitar migração de enum a cada novo estado operacional — decisão de implementação, não discutida na reunião.

### Criação do evento na outbox

Local: `src/modules/orders/order.service.ts`, dentro de `OrderService.changeStatus` (o mesmo bloco `this.prisma.$transaction(async (tx) => {...})` que já existe entre as linhas 131 e 178).

```
1. changeStatus valida a transição (canTransition) e aplica débito/reposição de estoque, como hoje.
2. Após tx.order.update(...) e tx.orderStatusHistory.create(...) (já existentes), chamar:
     await publishWebhookEvent(tx, order, from, to, userId)
3. publishWebhookEvent (nova função pura em src/modules/webhooks/webhook.publisher.ts):
   a. SELECT webhooks WHERE customerId = order.customerId AND active = true
      AND JSON_CONTAINS(events, to) -- filtra por status de destino
   b. Se nenhum webhook casar, retorna sem inserir nada (`[09:34]`, Bruno).
   c. Para cada webhook casado, monta o payload (ver "Contratos Públicos > Payload de Webhook")
      e insere um WebhookOutboxEvent com status = "PENDING", attempts = 0,
      nextAttemptAt = now().
4. Se o passo 3 lançar, toda a transação (incluindo a mudança de status) sofre rollback —
   é exatamente essa garantia que motiva o Outbox (ADR-001).
```

### Processamento pelo worker

Local: novo `src/worker.ts`, com lógica em `src/modules/webhooks/webhook.processor.ts`. Loop de polling a cada 2000ms (`setInterval` ou loop `while` com `await sleep(2000)` — equivalente; a reunião não especifica qual, ambos atendem ao requisito de 2s).

```
A cada tick (a cada 2s):
1. SELECT até 50 WebhookOutboxEvent
   WHERE status IN ('PENDING') AND nextAttemptAt <= now()
   ORDER BY createdAt ASC
   LIMIT 50
   (tamanho de lote de 50 é parâmetro de implementação, não definido na reunião —
    ajustável via variável de ambiente se necessário)
2. Para cada evento, em sequência dentro do worker (garante ordering por order_id
   discutida em ADR-007, já que um único worker processa em ordem de created_at):
   a. UPDATE WebhookOutboxEvent SET status = 'PROCESSING' WHERE id = ? AND status = 'PENDING'
      (guarda otimista: se 0 linhas afetadas, outro processo já pegou o evento — pular)
   b. Carregar o Webhook (secret ativa) correspondente.
   c. Computar assinatura: hex(HMAC-SHA256(secret, JSON.stringify(payload)))
   d. Disparar POST para webhook.url com timeout de 10s (AbortController + fetch nativo do Node 20).
   e. Se resposta 2xx dentro de 10s:
      - status = 'DELIVERED'
      - INSERT WebhookDelivery { outcome: 'SUCCESS', httpStatus, durationMs }
   f. Se erro de rede, timeout, ou resposta não-2xx:
      - INSERT WebhookDelivery { outcome: 'FAILURE', httpStatus (se houver), durationMs }
      - segue para "Retry" abaixo.
```

### Retry

Tabela de backoff fixa (ADR-003): `[1min, 5min, 30min, 2h, 12h]`, indexada por `attempts` (0 a 4).

```
Ao falhar (passo 2.f acima):
1. attempts += 1
2. Se attempts <= 5:
     nextAttemptAt = now() + BACKOFF[attempts - 1]
     status = 'PENDING' (volta pra fila)
3. Se attempts > 5:
     status = 'FAILED'
     move para Dead Letter (ver abaixo)
```

`BACKOFF` é uma constante `const RETRY_SCHEDULE_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]` em `webhook.processor.ts`, espelhando a mesma progressão documentada em ADR-003.

### Dead Letter Queue (DLQ)

```
Ao esgotar as 5 tentativas:
1. INSERT WebhookDeadLetter { outboxEventId, webhookId, payload, failureReason, failedAt: now() }
2. WebhookOutboxEvent correspondente permanece com status = 'FAILED' (não é deletado,
   fica como histórico bruto; WebhookDeadLetter é a fonte de verdade operacional).

Replay manual (POST /api/v1/admin/webhooks/dead-letter/:id/replay, role ADMIN):
1. Carrega o WebhookDeadLetter pelo id. Se não existir: WEBHOOK_DEAD_LETTER_NOT_FOUND (404).
2. Se replayedAt já preenchido: WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED (409)
   (guarda contra replay duplicado — decisão de implementação, não da reunião).
3. INSERT novo WebhookOutboxEvent com o mesmo payload, status = 'PENDING', attempts = 0.
4. UPDATE WebhookDeadLetter SET replayedAt = now().
5. logger.info({ requestId, adminUserId: req.user.id, deadLetterId, newOutboxEventId },
   'webhook_dead_letter_replayed')
   — resolve a questão em aberto RFC-Q-04 (mecanismo de auditoria do replay, não
   especificado na reunião): log estruturado via Pino, correlacionado por requestId
   e adminUserId, reaproveitando o padrão de log já usado em toda a aplicação
   (`src/shared/logger/index.ts`, `src/middlewares/request-logger.middleware.ts`).
```

## Contratos Públicos

Todas as rotas abaixo entram sob o prefixo `/api/v1` já aplicado em `src/app.ts:67` (`app.use('/api/v1', buildApiRouter(controllers))`) e exigem `Authorization: Bearer <token>` via `authenticate` (`src/middlewares/auth.middleware.ts`), como todo o resto da API. O CRUD de configuração aceita qualquer role autenticada; apenas o replay de DLQ exige `ADMIN` (`[09:36–09:37]`).

### `POST /api/v1/webhooks`

Cadastra um webhook. `customerId` vem no corpo (não no JWT — `[09:32–09:33]`), seguindo o mesmo padrão já usado em `CreateOrderInput.customerId` (`src/modules/orders/order.schemas.ts`).

Request:
```json
{
  "customerId": "8f14e45f-ceea-4c78-8b1c-1cb599a3a4b1",
  "url": "https://api.atlascomercial.com/webhooks/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:
```json
{
  "id": "b3f1e2a0-1234-4a5b-9c0d-abcdef012345",
  "customerId": "8f14e45f-ceea-4c78-8b1c-1cb599a3a4b1",
  "url": "https://api.atlascomercial.com/webhooks/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_2f8a1c...9d3e",
  "createdAt": "2026-07-19T14:00:00.000Z"
}
```
A `secret` só aparece em texto plano nesta resposta e na resposta de rotação (ver abaixo) — nunca em `GET`/`PATCH`, seguindo a exigência de proteção de secret de Sofia (`[09:21–09:22]`).

Erros: `400 WEBHOOK_INVALID_URL` (URL não-https, `[09:23]`), `404 WEBHOOK_CUSTOMER_NOT_FOUND`.

### `GET /api/v1/webhooks?customerId=...&page=1&pageSize=20`

Lista webhooks de um cliente, paginado no mesmo formato já usado por `CustomerService.list`/`ProductService.list` (`src/shared/http/response.ts`).

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "b3f1e2a0-1234-4a5b-9c0d-abcdef012345",
      "customerId": "8f14e45f-ceea-4c78-8b1c-1cb599a3a4b1",
      "url": "https://api.atlascomercial.com/webhooks/oms",
      "events": ["PAID", "SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-19T14:00:00.000Z",
      "updatedAt": "2026-07-19T14:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
`secret` nunca aparece nesta resposta.

### `PATCH /api/v1/webhooks/:id`

Edita `url`, `events` e/ou `active`.

Request:
```json
{ "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"], "active": true }
```

Response `200 OK`: mesmo formato de um item de `GET /api/v1/webhooks` (sem `secret`).

Erros: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL` (se `url` for enviada e não for https).

### `DELETE /api/v1/webhooks/:id`

Remove o cadastro. Response `204 No Content`. Erro: `404 WEBHOOK_NOT_FOUND`.

### `POST /api/v1/webhooks/:id/secret/rotate`

Gera nova secret; a anterior permanece válida por 24h (ADR-004).

Response `200 OK`:
```json
{
  "id": "b3f1e2a0-1234-4a5b-9c0d-abcdef012345",
  "secret": "whsec_9a7c3f...1e08",
  "previousSecretValidUntil": "2026-07-20T14:00:00.000Z"
}
```
Erro: `404 WEBHOOK_NOT_FOUND`.

### `GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=20`

Histórico das entregas mais recentes (até 100 por padrão de paginação — `[09:34]`).

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "d1e2f3a4-5678-4abc-9def-0123456789ab",
      "outcome": "FAILURE",
      "httpStatus": 503,
      "responseSnippet": "Service Unavailable",
      "durationMs": 10004,
      "attemptedAt": "2026-07-19T14:05:00.000Z"
    },
    {
      "id": "c9d8e7f6-4321-4abc-9def-0123456789ab",
      "outcome": "SUCCESS",
      "httpStatus": 200,
      "responseSnippet": "OK",
      "durationMs": 312,
      "attemptedAt": "2026-07-19T14:00:02.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 2, "totalPages": 1 }
}
```
Erro: `404 WEBHOOK_NOT_FOUND`.

### `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

Reprocessa manualmente um evento da DLQ (`[09:18–09:19]`). Exige role `ADMIN` (`requireRole('ADMIN')`, `src/middlewares/auth.middleware.ts`).

Response `202 Accepted`:
```json
{
  "deadLetterId": "e5f6a7b8-1111-4abc-9def-0123456789ab",
  "newOutboxEventId": "f6a7b8c9-2222-4abc-9def-0123456789ab",
  "status": "PENDING"
}
```
Erros: `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`, `403 FORBIDDEN` (role diferente de ADMIN — reaproveita `ForbiddenError` genérico já existente, não um código `WEBHOOK_*`, seguindo ADR-006).

### Payload de Webhook (entrega ao cliente)

Enviado pelo worker ao `url` cadastrado, não é uma rota da nossa API:

Headers:
```
Content-Type: application/json
X-Event-Id: f6a7b8c9-2222-4abc-9def-0123456789ab
X-Webhook-Id: b3f1e2a0-1234-4a5b-9c0d-abcdef012345
X-Signature: sha256=7d8a1e...c2f9
X-Timestamp: 2026-07-19T14:00:02.000Z
```

Body (`[09:43]`):
```json
{
  "event_id": "f6a7b8c9-2222-4abc-9def-0123456789ab",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-19T14:00:02.000Z",
  "order_id": "3c9e1f2a-aaaa-4abc-9def-0123456789ab",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "8f14e45f-ceea-4c78-8b1c-1cb599a3a4b1",
  "total_cents": 15990
}
```
`X-Signature` é `sha256=` + HMAC-SHA256 hex do corpo exato (string serializada antes do envio), usando a secret ativa do webhook — se dentro do grace period de rotação, o worker tenta a secret atual e, se o cliente não validar, isso é responsabilidade do cliente migrar dentro das 24h (o servidor não reenvia com a secret antiga automaticamente).

## Matriz de Erros (`WEBHOOK_*`)

| Código | HTTP | Descrição | Onde é lançado | Tratamento |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Webhook cadastrado não existe para o id informado | `WebhookService` (get/update/delete/rotate/deliveries) | Retornar 404 ao cliente da API |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` informado na criação não existe | `WebhookService.create`, segue o padrão de `OrderService.create` validando `tx.customer.findUnique` | Retornar 404 ao cliente da API |
| `WEBHOOK_INVALID_URL` | 400 | URL não é `https://` | Validação de schema Zod (`webhook.schemas.ts`), mesmo padrão de `src/modules/orders/order.schemas.ts` | Retornar 400 com detalhe do campo |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Tentativa de assinar/entregar evento para um webhook sem secret ativa (estado inconsistente, ex.: falha de migração de dados) | `webhook.processor.ts`, antes de computar HMAC | Não envia a chamada HTTP; evento vai direto para retry/DLQ com essa `failureReason` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload serializado ultrapassa 64KB (`[09:23–09:24]`) | `webhook.processor.ts`, antes de disparar o `fetch` | Não envia; evento vai para DLQ imediatamente (não é elegível a retry — payload não muda entre tentativas) |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno) | Chamada HTTP ao cliente não respondeu em 10s (`[09:42]`) | `webhook.processor.ts` | Registrado como `WebhookDelivery.outcome = FAILURE`; segue fluxo de retry |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Id de DLQ inexistente no endpoint de replay | `WebhookController.replayDeadLetter` | Retornar 404 |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | Replay solicitado para um evento já reprocessado | `WebhookController.replayDeadLetter` | Retornar 409 |

Todos os códigos acima seguem o padrão já existente no projeto: classes que estendem `AppError` (`src/shared/errors/app-error.ts`), análogas a `InsufficientStockError`/`InvalidStatusTransitionError` (`src/shared/errors/http-errors.ts`), capturadas sem alteração pelo middleware central (`src/middlewares/error.middleware.ts`) — ADR-006. Erros genéricos (`NOT_FOUND`, `FORBIDDEN`, `VALIDATION_ERROR`) continuam sendo usados quando não há semântica de negócio específica do módulo de webhooks (ex.: falha de autorização de role).

## Estratégias de Resiliência

- **Timeout**: 10 segundos por chamada HTTP de entrega (`AbortController`), decidido em `[09:42]` para equilibrar tolerância a clientes lentos sem travar o worker indefinidamente em uma única chamada.
- **Retry e backoff**: exponencial, 5 tentativas, progressão fixa `1min/5min/30min/2h/12h` (ADR-003). Implementado via `nextAttemptAt` no próprio `WebhookOutboxEvent`, sem fila de retry separada.
- **Fallback**: após esgotar as tentativas, o evento vai para `WebhookDeadLetter` — nenhum evento é descartado silenciosamente; reprocessamento é manual via endpoint admin (ADR-003).
- **Guarda de concorrência**: `UPDATE ... WHERE status = 'PENDING'` no passo de tomar um evento (seção "Processamento pelo worker") evita processamento duplicado caso, no futuro, mais de um worker rode simultaneamente — mitigação defensiva; hoje a topologia é single-worker (ADR-007) e essa guarda não é estritamente necessária, mas é barata e evita reintrodução silenciosa do problema se alguém subir um segundo worker sem revisitar ADR-007.
- **Rate limiting / circuit breaker**: não implementado nesta fase — item em aberto explicitamente adiado na reunião (RFC-Q-01, `[09:38–09:39]`).

## Observabilidade

### Métricas

- `webhook_outbox_pending_total` (gauge) — eventos aguardando processamento; sinaliza acúmulo/atraso do worker.
- `webhook_deliveries_total{outcome="success"|"failure"}` (counter) — volume de entregas por resultado.
- `webhook_delivery_duration_ms` (histograma) — latência das chamadas HTTP de entrega, para acompanhar aproximação do timeout de 10s.
- `webhook_dead_letter_total` (counter) — eventos que esgotaram retry, principal sinal de saúde da integração com cada cliente.
- `webhook_dead_letter_replays_total` (counter) — volume de replays manuais executados por operadores.

Nenhuma dessas métricas existe hoje no projeto (`ANALIZE_CODEBASE.md` não identifica infraestrutura de métricas); ficam como especificação para a implementação, não como extensão de algo já existente.

### Logs

Reaproveita o logger Pino já configurado em `src/shared/logger/index.ts` (mesmo `redact` de campos sensíveis — a secret do webhook deve ser adicionada à lista `redactPaths` nesse arquivo). Eventos-chave a logar, todos como log estruturado (`logger.info`/`logger.warn`/`logger.error`), com `webhookId`, `outboxEventId` e `orderId` como campos de correlação:

- `webhook_event_published` — ao inserir na outbox (dentro de `publishWebhookEvent`).
- `webhook_delivery_attempted` — a cada tentativa do worker, com `attempts`, `outcome`, `durationMs`.
- `webhook_delivery_exhausted` — ao mover para DLQ.
- `webhook_dead_letter_replayed` — ao processar o endpoint de replay (ver seção "Dead Letter Queue" acima), com `adminUserId` e `requestId` para auditoria.

### Tracing

O projeto não possui hoje nenhuma infraestrutura de tracing distribuído (`ANALIZE_CODEBASE.md`). Para esta feature, a correlação entre a requisição HTTP que mudou o status do pedido, a inserção na outbox e a entrega final é feita por IDs propagados manualmente nos logs (`orderId` → `outboxEventId`/`event_id` → `X-Event-Id` no request enviado ao cliente), reaproveitando o padrão de `requestId` já usado em `src/middlewares/request-logger.middleware.ts`. Instrumentação de tracing formal (ex.: OpenTelemetry) fica como recomendação futura, fora do escopo desta implementação — limitação explícita, não uma decisão da reunião.

## Dependências e Compatibilidade

- **Banco de dados**: MySQL já existente via Prisma; quatro modelos novos exigem uma migration (`prisma migrate dev`), sem alteração em modelos existentes.
- **Cliente HTTP**: nenhuma dependência nova — Node 20 (`package.json` já exige `"node": ">=20"`) inclui `fetch` nativo (undici), suficiente para as chamadas de entrega com `AbortController` para o timeout.
- **HMAC**: módulo nativo `node:crypto`, sem dependência nova.
- **UUID**: reaproveita o pacote `uuid` já presente em `package.json`.
- **Processo do worker**: novo script `"worker": "tsx watch --env-file=.env src/worker.ts"` em `package.json`, espelhando o script `"dev"` já existente para `src/server.ts`.
- **Variáveis de ambiente**: nenhuma nova variável obrigatória identificada na reunião; o schema Zod em `src/config/env.ts` deve ser estendido apenas se parâmetros como tamanho de lote do worker forem externalizados (não decidido).
- **Compatibilidade**: mudança aditiva — nenhum contrato de API existente (`orders`, `products`, `customers`, `users`, `auth`) é alterado. `OrderService.changeStatus` ganha uma chamada adicional dentro de uma transação já existente; sua assinatura pública não muda.

## Critérios de Aceite Técnicos

- Mudar o status de um pedido para um status coberto por pelo menos um webhook ativo do cliente insere exatamente um `WebhookOutboxEvent` por webhook casado, na mesma transação — testável simulando falha de inserção da outbox e verificando rollback também da mudança de status.
- Nenhum `WebhookOutboxEvent` é criado quando não há webhook ativo com aquele status no filtro (`[09:34]`).
- O worker processa um evento `PENDING` com `nextAttemptAt <= now()` em até um ciclo de polling (≤2s) após ele se tornar elegível.
- Uma entrega que responde 2xx em menos de 10s resulta em `status = DELIVERED` e um `WebhookDelivery` com `outcome = SUCCESS`.
- Uma entrega que falha define corretamente `nextAttemptAt` conforme a tabela de backoff, e após a 5ª falha o evento aparece em `WebhookDeadLetter` com `WebhookOutboxEvent.status = FAILED`.
- `X-Signature` calculado pelo cliente de teste com a secret retornada na criação bate byte a byte com o corpo recebido.
- Rotação de secret: a secret antiga continua validando assinaturas por 24h após a rotação; após esse prazo, deixa de ser aceita (teste com relógio controlado/mocado).
- Endpoint de replay: `403` para usuário não-ADMIN, `202` e novo `WebhookOutboxEvent` `PENDING` para ADMIN, `409` em replay duplicado.
- Suite de testes segue o padrão de `tests/orders.test.ts` (Vitest + Supertest + `tests/helpers/factories.ts`), com um novo `tests/webhooks.test.ts` e uma nova factory `createTestWebhook` em `tests/helpers/factories.ts`.

## Riscos e Mitigação

- **Migração do schema em produção sem downtime.** Quatro tabelas novas, nenhuma alteração em tabela existente — migration aditiva de baixo risco, aplicável com o pipeline `prisma migrate` já em uso.
- **Payload de teste divergir do payload real por mudança futura em `Order`.** Mitigação: `publishWebhookEvent` lê os campos diretamente do objeto `order` já carregado na transação de `changeStatus`, não duplica lógica de serialização em outro lugar.
- **Worker ficar para trás sob volume alto (mitiga parcialmente o risco RFC-RISK-02 de gargalo em worker único).** Mitigação nesta fase: lote de leitura configurável (hoje fixo em 50) e métrica `webhook_outbox_pending_total` para detectar acúmulo cedo; escalar para múltiplos workers é explicitamente fora de escopo (ADR-007).
- **Secret em texto plano no banco.** Necessário porque HMAC exige a secret disponível para assinar (não é possível usar hash irreversível como para senhas de usuário, `bcrypt` em `src/modules/users/user.service.ts`). Mitigação: campo já sinalizado para `redact` em logs (seção Observabilidade); acesso ao banco de produção deve seguir os controles de acesso já existentes na infraestrutura da empresa (fora do escopo desta feature).

## Integração com o Sistema Existente

- **`src/modules/orders/order.service.ts`** — `OrderService.changeStatus` (linhas 126-179) ganha uma chamada a `publishWebhookEvent(tx, order, from, to, userId)` logo após `tx.orderStatusHistory.create`, dentro do mesmo `prisma.$transaction`. Nenhuma outra parte do método muda.
- **`src/app.ts`** — `buildControllers` (linhas 26-53) ganha a construção de `WebhookRepository`, `WebhookService` e `WebhookController`, seguindo exatamente o padrão já usado para `orderRepository`/`orderService`/`orderController`; o objeto `Controllers` retornado passa a incluir `webhooks`.
- **`src/routes/index.ts`** — `buildApiRouter` ganha `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` e `router.use('/admin/webhooks', buildAdminWebhookRouter(controllers.webhooks))`, no mesmo padrão dos `router.use(...)` já existentes para `orders`, `products`, `customers`.
- **`src/shared/errors/http-errors.ts`** (ou um novo `src/modules/webhooks/webhook.errors.ts` reexportado por `src/shared/errors/index.ts`) — novas classes `WebhookNotFoundError`, `WebhookInvalidUrlError` etc., todas estendendo `AppError`, no mesmo molde de `InsufficientStockError`/`InvalidStatusTransitionError`.
- **`src/middlewares/auth.middleware.ts`** — `requireRole('ADMIN')` (já usado em `src/modules/users/user.routes.ts:15` para `GET /users/:id`) é reaproveitado sem alteração na rota `POST /admin/webhooks/dead-letter/:id/replay`.
- **`src/shared/http/response.ts`** — `paginated()` é reaproveitado sem alteração para `GET /webhooks` e `GET /webhooks/:id/deliveries`, no mesmo formato já usado por `CustomerService.list`/`ProductService.list`.
- **`src/shared/logger/index.ts`** — o array `redactPaths` (linha 4) precisa ganhar uma entrada para o campo `secret`/`previousSecret` do modelo `Webhook`, seguindo o padrão já usado para `*.password`/`*.passwordHash`.
- **`src/server.ts`** — padrão de bootstrap (criação do `PrismaClient` via `createPrismaClient()` de `src/config/database.ts`, listen, shutdown gracioso em `SIGINT`/`SIGTERM`) é replicado em `src/worker.ts`, substituindo `app.listen(...)` pelo loop de polling do worker; `prisma.$disconnect()` no shutdown continua igual.
- **`prisma/schema.prisma`** — quatro modelos novos (`Webhook`, `WebhookOutboxEvent`, `WebhookDelivery`, `WebhookDeadLetter`), sem alteração nos modelos existentes; nova migration em `prisma/migrations/`.
- **`package.json`** — novo script `"worker"`, espelhando `"dev"` (linha 11), e nenhuma dependência nova (ver "Dependências e Compatibilidade").
