# Análise da Base de Código

Exploração somente leitura da base de código (`src/`, `prisma/schema.prisma`) para mapear módulos existentes, a máquina de estados do pedido, o controle de estoque, a auditoria de mudanças de status e os padrões de erro/exceção utilizados.

## 1. Visão geral da arquitetura

Aplicação Node.js/TypeScript com Express e Prisma (MySQL), organizada em módulos de domínio sob `src/modules/`, com componentes compartilhados em `src/shared/` e `src/middlewares/`.

- **Entrada HTTP**: `src/app.ts` monta o Express, aplica `requestLogger` e `errorMiddleware`, e registra as rotas via `src/routes/index.ts` (`buildApiRouter`).
- **Módulos de domínio**: `auth`, `users`, `customers`, `products`, `orders` — cada um com o padrão `*.routes.ts` → `*.controller.ts` → `*.service.ts` → `*.repository.ts` (mais `*.schemas.ts` para validação com Zod).
- **Persistência**: Prisma ORM, schema único em `prisma/schema.prisma`, migrações em `prisma/migrations/`.
- **Compartilhado**: `src/shared/errors/` (hierarquia de exceções de domínio), `src/shared/http/response.ts` (paginação), `src/shared/logger/` (logger estruturado).

## 2. Módulos de domínio

| Módulo | Entidades | Papel |
|---|---|---|
| `auth` | `User` (via `UserRepository`) | Login (JWT) e registro. Não define modelo próprio; delega criação de usuário ao `UserService`. |
| `users` | `User` | CRUD de usuários, hashing de senha (bcrypt), papéis (`ADMIN`, `OPERATOR`). |
| `customers` | `Customer` | CRUD de clientes; e-mail único; endereço armazenado como `Json`. |
| `products` | `Product` | CRUD de produtos/catálogo; controla `stockQuantity` e `active`. |
| `orders` | `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence` | Módulo central: criação de pedidos, transição de status, débito/reposição de estoque, histórico de auditoria. |

**Relações entre módulos:**
- `orders` depende de `customers` (cliente do pedido) e `products` (itens, preço, estoque).
- `orders` depende de `users` (quem criou o pedido, quem mudou o status) via `createdById` e `changedById`.
- `auth` depende de `users` para autenticação/registro.

Autorização por papel (`requireRole`) só foi encontrada no módulo `users` (rota de criação restrita a `ADMIN` — ver `src/modules/users/user.routes.ts:15`). As demais rotas de escrita (`orders`, `products`, `customers`) exigem apenas autenticação (`authenticate`), sem restrição adicional de papel — ponto a confirmar como intencional ou lacuna.

## 3. Máquina de estados do pedido

Definida em `src/modules/orders/order.status.ts`, com enum `OrderStatus` no schema Prisma.

**Estados**: `PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, com `CANCELLED` como saída alternativa a partir de `PENDING`, `PAID` ou `PROCESSING`.

**Transições permitidas** (`transitions` em `order.status.ts:3-10`):

```
PENDING     → PAID, CANCELLED
PAID        → PROCESSING, CANCELLED
PROCESSING  → SHIPPED, CANCELLED
SHIPPED     → DELIVERED
DELIVERED   → (terminal)
CANCELLED   → (terminal)
```

- Estado inicial: `PENDING` (default no schema e na criação do pedido).
- Estados terminais: `DELIVERED` e `CANCELLED` (`isTerminal()`).
- `SHIPPED` só pode ir para `DELIVERED` — não pode ser cancelado a partir daí.
- Transição para o mesmo status é bloqueada explicitamente (`ConflictError` em `order.service.ts:140-146`, antes mesmo de checar `canTransition`).

**Onde é aplicada:** inteiramente em código de aplicação (`OrderService.changeStatus`, `src/modules/orders/order.service.ts:126-179`), não há `CHECK` constraint no banco. O Prisma/MySQL só garante que o valor pertence ao enum `OrderStatus`; a validade da transição é responsabilidade da camada de serviço.

Fluxo de `changeStatus`:
1. Carrega o pedido com itens dentro de uma transação (`prisma.$transaction`).
2. Rejeita se `from === to` (`ConflictError` / `INVALID_STATUS_TRANSITION`).
3. Rejeita se a transição não é permitida (`InvalidStatusTransitionError`, HTTP 409).
4. Aplica efeitos colaterais de estoque conforme a transição (ver seção 4).
5. Atualiza `order.status` e insere um registro em `OrderStatusHistory`.
6. Retorna o pedido atualizado com relações.

## 4. Controle de estoque

Campo `stockQuantity` em `Product` (schema.prisma:62). Não há tabela de reserva separada nem campo `reserved`; o estoque é debitado/reposto diretamente sobre o produto.

**Regras de disparo** (`order.status.ts:24-37`):
- `shouldDebitStock`: débito ocorre exatamente na transição `PENDING → PAID`.
- `shouldReplenishStock`: reposição ocorre ao cancelar a partir de `PAID` ou `PROCESSING` (transição para `CANCELLED`).
- Nenhum efeito de estoque nas transições `PROCESSING → SHIPPED` ou `SHIPPED → DELIVERED` (o estoque já foi debitado no pagamento).
- Cancelamento a partir de `PENDING` não repõe estoque, pois nesse ponto o estoque ainda não foi debitado.

**Implementação** (`OrderService.debitStock` / `replenishStock`, `order.service.ts:204-243`):
- Executado dentro da mesma transação Prisma da mudança de status (`tx`), garantindo atomicidade entre validação de status, débito/reposição de estoque e gravação do histórico.
- `debitStock` primeiro busca todos os produtos envolvidos, calcula indisponibilidade (`product.stockQuantity < item.quantity`) e lança `InsufficientStockError` (HTTP 422) se houver qualquer item sem estoque suficiente — falha atômica: nenhum produto é debitado se algum item falhar.
- Débito/reposição usam operadores atômicos do Prisma (`{ decrement }` / `{ increment }`), evitando leitura-e-escrita não atômica a nível de linha.
- **Lacuna de concorrência**: a checagem de disponibilidade (`findMany`) e o decremento (`update`) não estão protegidos por lock explícito (`SELECT ... FOR UPDATE`) nem por um campo de versão (`@@version`). Em MySQL, sem isolamento `SERIALIZABLE` explícito, duas transações concorrentes que leem o mesmo estoque antes de decrementar podem, em teoria, ambas passar a checagem antes de qualquer commit, gerando overselling sob alta concorrência. Isso depende do nível de isolamento padrão do MySQL (`REPEATABLE READ`) e não foi possível confirmar mitigação adicional no código.
- Na criação do pedido (`OrderService.create`), **não há débito de estoque** — apenas os produtos são validados como existentes e ativos. O débito só acontece ao mover para `PAID`. Ou seja, é possível criar um pedido `PENDING` com quantidade acima do estoque disponível; a checagem de estoque só ocorre no pagamento.

## 5. Auditoria de mudanças de status

Tabela dedicada `OrderStatusHistory` (`schema.prisma:116-131`), mapeada para `order_status_history`.

- Campos: `fromStatus` (nulo na criação), `toStatus`, `changedAt`, `changedById` (FK para `User`), `reason` (opcional, texto livre até 500 caracteres).
- Um registro é criado tanto na criação do pedido (`fromStatus: null`, `toStatus: PENDING`, `reason: 'order created'`) quanto em cada `changeStatus` subsequente, sempre dentro da mesma transação da mudança de status — garante que histórico e status atual nunca fiquem dessincronizados por falha parcial.
- Consulta: `history: { orderBy: { changedAt: 'asc' } }` é incluída em `findByIdWithRelations` e nas respostas de `create`/`changeStatus`, permitindo reconstruir a linha do tempo completa do pedido a qualquer momento.
- Não há auditoria equivalente para mudanças em `Product` (ex.: alterações de `stockQuantity` fora do fluxo de pedidos, ou mudanças de preço) nem para `Customer`. A rastreabilidade de auditoria no sistema está restrita ao ciclo de vida do pedido.
- Log de aplicação (`request-logger.middleware.ts`) registra todas as requisições HTTP (método, path, status, duração, `userId`) via logger estruturado, mas isso é um log operacional, não um mecanismo de auditoria de domínio.

## 6. Padrões de erro e exceção

Hierarquia centralizada em `src/shared/errors/`:

- `AppError` (base): `message`, `statusCode`, `errorCode`, `details` opcionais.
- Subclasses HTTP genéricas: `BadRequestError` (400), `ValidationError` (400, `VALIDATION_ERROR`), `UnauthorizedError` (401), `ForbiddenError` (403), `NotFoundError` (404, mensagem `"${resource} not found"`), `ConflictError` (409), `UnprocessableEntityError` (422).
- Subclasses de domínio específicas, que herdam de `ConflictError`/`UnprocessableEntityError` e carregam `details` estruturados:
  - `InvalidStatusTransitionError` (409, `INVALID_STATUS_TRANSITION`, `{ from, to }`).
  - `InsufficientStockError` (422, `INSUFFICIENT_STOCK`, `{ unavailable: [{ sku, requested, available }] }`).

**Origem dos erros**: majoritariamente lançados na camada de serviço (`*.service.ts`), após validações de regra de negócio (existência, unicidade, estado válido). Os repositórios não lançam `AppError` — apenas retornam `null`/lançam erros do Prisma, que o serviço traduz.

**Tratamento centralizado** (`src/middlewares/error.middleware.ts`):
1. `AppError` → resposta JSON `{ error: { code, message, details? } }` com o `statusCode` da própria instância.
2. `ZodError` (validação de schema não capturada antes) → 400, `VALIDATION_ERROR`, com `details` formatado a partir de `issue.path`/`issue.message`.
3. `Prisma.PrismaClientKnownRequestError`:
   - `P2002` (violação de unicidade) → 409, `CONFLICT`.
   - `P2025` (registro não encontrado) → 404, `NOT_FOUND`.
4. Qualquer outro erro não mapeado → log estruturado via `logger.error` (com `requestId`, método, path) e resposta genérica 500, `INTERNAL_SERVER_ERROR` — sem vazar detalhes internos ao cliente.

**Validação de entrada**: `src/middlewares/validate.middleware.ts` (não lido em detalhe, mas referenciado em todas as rotas) aplica schemas Zod a `params`/`query`/`body` antes do controller, cortando a maior parte dos erros de formato antes de chegar à camada de serviço.

**Padrão geral**: fail-fast, sem retries nem compensação automática — toda falha de regra de negócio interrompe a transação Prisma (`$transaction`), que faz rollback automático, mantendo consistência entre pedido, estoque e histórico.

## 7. Lacunas e pontos em aberto

- **Concorrência no débito de estoque**: sem lock explícito ou campo de versão; risco teórico de overselling sob alta concorrência (ver seção 4).
- **Checagem de estoque tardia**: pedidos podem ser criados com quantidade além do estoque disponível; a validação só ocorre no momento do pagamento (`PENDING → PAID`).
- **Autorização por papel inconsistente**: `requireRole('ADMIN')` só é usado na criação de usuários; não há evidência de controle de papel em `orders`, `products` ou `customers` — a decidir se é intencional.
- **Sem auditoria fora de pedidos**: mudanças em `Product` (preço, estoque, ativação) e `Customer` não têm histórico equivalente ao `OrderStatusHistory`.
- **`validate.middleware.ts` não foi inspecionado em detalhe** — apenas seu uso nas rotas foi confirmado.

## Observações sobre a exploração

- Overrides aplicados: nenhum além do escopo já solicitado (mapear módulos, máquina de estados, estoque, auditoria, erros).
- Leitura direcionada: arquivos centrais de cada módulo (`service`, `repository`, `status`, schema Prisma, middlewares de erro/autenticação/log) foram lidos integralmente; `*.controller.ts`, `*.schemas.ts` e `validate.middleware.ts` foram inferidos por uso nas rotas, não lidos linha a linha.
- Nenhum arquivo foi modificado; análise puramente descritiva.
