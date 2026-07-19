# Reuso dos padrões existentes do projeto

- Status: Aceito
- Data: 2026-07-19 (data de registro deste ADR; a decisão foi tomada na reunião técnica descrita em `TRANSCRICAO.md`)
- Deciders: Bruno (Engenheiro Pleno, Pedidos), Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead)
- Relacionado: `TRANSCRICAO.md` [09:27–09:30]; `ANALIZE_CODEBASE.md`

## Status

Aceito.

## Contexto

Ao discutir a estrutura de código do novo módulo, Bruno apontou que o projeto já segue um padrão claro: cada domínio vive em `src/modules/<dominio>` com `controller`, `service`, `repository`, `routes` e `schemas` (confirmado em `ANALIZE_CODEBASE.md`, presente hoje em `src/modules/auth`, `src/modules/users`, `src/modules/customers`, `src/modules/products` e `src/modules/orders`). Ele propôs que o módulo de webhooks siga exatamente essa estrutura, em `src/modules/webhooks` (`[09:27]`).

Sobre erros, Bruno lembrou que o projeto já tem uma hierarquia própria: a classe base `AppError` (`src/shared/errors/app-error.ts`) e subclasses específicas como `InsufficientStockError` e `InvalidStatusTransitionError` (`src/shared/errors/http-errors.ts`), cada uma com um código de erro estável (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`). Ele propôs seguir o mesmo padrão para o módulo de webhooks, com códigos prefixados por `WEBHOOK_` (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, entre outros) — proposta que Larissa confirmou como prefixo padrão do módulo (`[09:28–09:29]`).

Bruno também apontou que o logger Pino já está presente no projeto inteiro (`src/shared/logger/index.ts`) e que o middleware de erro centralizado (`src/middlewares/error.middleware.ts`) já trata `AppError`, `ZodError` e erros conhecidos do Prisma (`P2002`, `P2025`) sem exigir nenhuma alteração — os novos erros de webhook, por herdarem de `AppError`, serão automaticamente capturados por esse middleware (`[09:29]`). Diego levantou a questão do acesso ao banco pelo worker; Bruno confirmou que o pool/instância de conexão do Prisma seguiria o padrão já existente, com uma instância própria de `PrismaClient` por processo (ver ADR-002). Larissa fechou o ponto resumindo a decisão como "reuso máximo do que já existe: `AppError`, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro" (`[09:30]`).

## Decisão

O módulo de webhooks é implementado como mais um módulo de domínio em `src/modules/webhooks`, replicando a estrutura já existente em `src/modules/orders` (controller, service, repository, routes, schemas). Erros de domínio do módulo estendem a hierarquia já existente em `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`, seguindo o mesmo padrão de `InsufficientStockError`/`InvalidStatusTransitionError`, com códigos prefixados por `WEBHOOK_`. O módulo reaproveita, sem alteração:

- o logger Pino de `src/shared/logger/index.ts`;
- o middleware de erro centralizado `src/middlewares/error.middleware.ts`, que já mapeia `AppError` (e suas novas subclasses `WEBHOOK_*`), `ZodError` e erros conhecidos do Prisma para respostas HTTP consistentes;
- o mecanismo de autorização `requireRole` de `src/middlewares/auth.middleware.ts`, usado para restringir o endpoint administrativo de replay da DLQ (ADR-003) à role `ADMIN`;
- o padrão de validação de entrada com schemas Zod, já usado em todos os módulos existentes (ex.: `src/modules/orders/order.schemas.ts`).

## Alternativas Consideradas

- **Criar uma hierarquia de erros e um logger próprios para o módulo de webhooks**, isolados do restante do projeto. Rejeitada implicitamente pela equipe: fragmentaria o tratamento de erros e a observabilidade da aplicação em dois padrões distintos, sem nenhum ganho identificado na reunião, e obrigaria o middleware de erro centralizado a conhecer um segundo formato de erro (`[09:28–09:29]`).

## Consequências

### Positivas

- Reduz drasticamente a superfície de código novo: o middleware de erro, o logger e o mecanismo de autorização já existentes funcionam para o módulo de webhooks sem nenhuma modificação.
- Mantém consistência de padrão de código para quem já trabalha na base — um desenvolvedor familiarizado com `src/modules/orders` reconhece imediatamente a estrutura de `src/modules/webhooks`.
- Códigos de erro `WEBHOOK_*` seguem a mesma convenção previsível já usada por `INSUFFICIENT_STOCK`/`INVALID_STATUS_TRANSITION`, facilitando o consumo desses erros tanto internamente quanto por quem monitora a aplicação.

### Negativas

- Herda também as limitações já presentes nesses padrões (por exemplo, o middleware de erro centralizado não distingue níveis de severidade além do status HTTP); qualquer ajuste futuro nesses mecanismos compartilhados passa a impactar todos os módulos, incluindo webhooks.
- Acopla o módulo de webhooks a decisões estruturais tomadas anteriormente para outros domínios (ex.: forma de organizar `repository`/`service`), mesmo que o domínio de entrega de webhooks tenha características distintas (processamento assíncrono via worker, não só request/response).
