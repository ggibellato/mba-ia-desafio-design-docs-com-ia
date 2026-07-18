---
name: analyze-codebase  
description: Explores a codebase to map its domain modules, state machines, data models, and error-handling patterns, producing a concise architectural report.
---

# Analyze Codebase

Systematic, read‑only exploration of an existing codebase to answer “what’s here and how does it behave?”. The skill orients itself in the repository, identifies key domain modules, infers state machines from code and schema, analyzes inventory and auditing logic, and summarizes error/exception patterns. [skillmp](https://skillmp.ru/skills/tools/ide-plugins/exploring-codebases-skill-dlya-analiza-kodovoy-bazy/)

## INPUT

Free‑form. The skill infers what was passed. Any combination works:

- A repo path: `.`, `/path/to/repo`, `apps/api/`, etc.  
- A subdirectory within a repo: `src/`, `backend/`, `services/orders/`.  
- A specific schema or config file: `prisma/schema.prisma`, `schema.sql`, `models.py`, `*.orm.xml`.  
- A domain hint: `orders and inventory`, `map auth/users/customers/products/orders`, `focus on error handling`.  
- Extra natural‑language instructions appended anywhere (see **Overrides**).

The skill tries to locate three types of context:

1. **Primary source roots** (code to inspect):  
   - Prefer typical directories: `src/`, `app/`, `backend/`, `services/`, `packages/*`.  
   - If input points to a file, use its parent folder as a starting root.  
   - If input is just a repo path, auto‑discover source roots by scanning for language‑specific entry points (e.g. `main.*`, `server.*`, `app.*`, `index.*`). [skillmp](https://skillmp.ru/skills/tools/ide-plugins/exploring-codebases-skill-dlya-analiza-kodovoy-bazy/)

2. **Data model/schema sources** (for state machine and inventory mapping):  
   - ORM schemas: `prisma/schema.prisma`, `*.prisma`, `models/*.py`, `entities/*.ts`, `schema.*`.  
   - Migrations: `prisma/migrations/`, `db/migrations/`, `sql/`.  
   - Direct database schema files: `schema.sql`, `ddl/*.sql`. [prisma](https://www.prisma.io/docs/llms-full.txt)

3. **Configuration and conventions** (for domain/module detection):  
   - Routing or controller trees: `routes/`, `controllers/`, `api/`.  
   - Auth/identity: `auth/`, `security/`, `users/`.  
   - Error handling: `errors/`, `exceptions/`, logging configs, middleware.

If nothing looks like source or schema under the given path, the skill aborts:  
> “No usable source or schema roots found under `<path>`. Pass a repo or source directory.”

## OUTPUT

A single **analysis report**, in markdown, with sections like:

- **Domain modules**: auth, users, customers, products, orders (or whatever exists in this repo), including main entry points and relationships.  
- **Order state machine**: states, transitions, invariants, and where they are enforced (code vs. schema vs. config).  
- **Inventory control**: how stock changes are modeled, when and where they’re updated, consistency strategies (transactions, locks, events).  
- **Status audit trail**: how status changes are recorded (audit tables, logs, event streams), and how traceability is implemented.  
- **Error/exception patterns**: common error types, how they’re surfaced to callers, logging and retry patterns.  
- **Unknowns and open questions**: areas that are ambiguous or under‑specified.

No code is modified. No commits are created. Output is purely descriptive.

***

## EXECUTION STEPS

### Step 1: Resolve Input

Parse the entire input as free‑form. Extract:

- **Repo or root reference**:  
  - First token that resolves to an existing directory.  
  - If none is explicit, assume `.` (current working directory).  
- **Focus hints**: phrases like `auth`, `users`, `customers`, `products`, `orders`, `inventory`, `state machine`, `exceptions`, `error handling`.  
- **Extra instructions**: remaining free text; treat as overrides (see Step 3).

If resolution fails (directory not found, or not a repo‑like tree):

- Abort with a clear message:  
  > “Could not resolve a codebase from `<input>`. Pass a repo path or source directory.”

### Step 2: Map Structure (High‑Level)

Run a structural scan to understand “what’s here”: languages, directories, file types. [skillmp](https://skillmp.ru/skills/tools/ide-plugins/exploring-codebases-skill-dlya-analiza-kodovoy-bazy/)

- Identify:
  - Root‑level directories and primary source roots (e.g. `src/`, `apps/api/`, `services/orders/`).  
  - Language mix (TypeScript, Java, Python, etc.).  
  - Density markers: directories with many symbols per file, many imports, or routing definitions. [skillmp](https://skillmp.ru/skills/tools/ide-plugins/exploring-codebases-skill-dlya-analiza-kodovoy-bazy/)

- Record:
  - Candidate **domain modules** by name (e.g. `auth`, `users`, `customers`, `products`, `orders`, `inventory`, `stock`, `billing`), based on directory names, file names, and class/function identifiers. [remoteopenclaw](https://www.remoteopenclaw.com/skills/404kidwiz/claude-supercode-skills/codebase-exploration)
  - Candidate **data model locations** (ORM schemas, migrations, models) for deeper analysis.

Do not read every file yet—this step is inventory and orientation.

If no meaningful source roots are found:

- Abort:  
  > “Codebase mapped, but no source roots suitable for analysis (only vendored/generated/docs). Narrow the path (e.g. `src/`, `backend/`).”

### Step 3: Apply Overrides

Interpret extra instructions as natural‑language overrides on defaults:

| Default | Example overrides |
|---|---|
| Thoroughness = medium | “quick skim only”, “go very deep” |
| Module scope = inferred | “only orders and inventory”, “skip auth” |
| Focus = domain + errors | “ignore error handling”, “focus on state machines” |
| Output language = input’s language | “output in English”, “output in Portuguese” |

For each recognized override, record that it was applied (for inclusion in the report’s “Overrides applied” section). Unclear or contradictory overrides → keep defaults and note under “Overrides ignored”.

Immutable core: the skill stays **read‑only** and must produce a structured report; overrides cannot request code changes.

### Step 4: Domain Module Discovery

Identify existing domain modules, using both structure and semantics. [skillmp](https://skillmp.ru/skills/tools/ide-plugins/exploring-codebases-skill-dlya-analiza-kodovoy-bazy/)

- Use directory and file names, namespace/package names, and common patterns to detect modules such as:
  - Auth / identity: `auth`, `security`, `sessions`, `tokens`.  
  - Users / customers: `users`, `customers`, `accounts`, `profiles`.  
  - Products / catalog: `products`, `catalog`, `items`, `sku`.  
  - Orders / sales: `orders`, `checkout`, `cart`, `fulfillment`.  
  - Inventory / stock: `inventory`, `stock`, `warehouse`.  

- For each detected module:
  - Locate key entry points (controllers, services, use‑cases, handlers).  
  - Map relationships (e.g. `orders` depends on `customers` and `products`, `inventory` updates triggered by order events).

Read files lazily: for each module, open representative core files (service/handler/model), not the entire tree, to confirm the module’s purpose and boundaries.

Record findings under **Domain Modules** in the report.

### Step 5: Data Model & State Machines

Drill into schema and model layers to extract state machines and inventory logic. [advanced-stack](https://advanced-stack.com/resources/how-to-extract-and-analyze-a-code-base-with-llms.html)

5.1 **Locate schemas**

- Look for:
  - Prisma schemas (`*.prisma`), especially `schema.prisma`.  
  - ORM models in code (e.g. `User`, `Customer`, `Product`, `Order`, `Inventory` classes).  
  - Database migrations or DDL (`migrations/`, `schema.sql`). [prisma](https://www.prisma.io/docs/llms-full.txt)

5.2 **Order state machine**

- For order‑like entities:
  - Enumerate status fields (`status`, `state`, `stage`) and their possible values (enum definitions, string literals, TypeScript unions, etc.).  
  - Identify transitions:
    - Code paths where the status changes (services, handlers, workflows).  
    - Preconditions and guards (payments, stock checks, validations).  
  - Note:
    - Initial state, terminal states.  
    - Forbidden or unusual transitions (constraints, validations, assertions).

Summarize as a state machine diagram in prose: states, transitions, triggers, and enforcement points (schema constraints vs. application code).

5.3 **Inventory control**

- For stock/inventory entities:
  - Identify fields like `stock`, `quantity`, `available`, `reserved`.  
  - Find update patterns:
    - Where stock is decremented/incremented (order placement, cancellation, returns).  
    - Whether updates are transactional, event‑driven, or best‑effort (e.g. async tasks).  
  - Look for consistency devices:
    - Transactions, row‑level locks, unique constraints, versioning fields (`version`, `updatedAt`). [prisma](https://www.prisma.io/docs/llms-full.txt)

Document:
- How inventory changes are triggered by order events.  
- How overselling is prevented (if at all).  
- Known race‑condition mitigations (locks/transactions) or lack thereof.

5.4 **Audit trail of status changes**

- Search for:
  - Audit tables (`OrderStatusHistory`, `AuditLog`, `EventLog`).  
  - Domain events (`OrderCreated`, `OrderShipped`, etc.).  
  - Logging and tracing middleware that captures status changes.  

Describe:
- Where status changes are recorded (DB vs. logs vs. events).  
- How they can be reconstructed (e.g. querying history tables, reading event streams, correlating logs).  
- Any gaps (status changes without clear audit traces).

### Step 6: Error and Exception Patterns

Analyze how errors are represented and handled across the codebase. [skillmp](https://skillmp.ru/skills/tools/ide-plugins/exploring-codebases-skill-dlya-analiza-kodovoy-bazy/)

- Identify:
  - Custom error/exception types (e.g. `DomainError`, `ValidationError`, `OutOfStockError`, `UnauthorizedError`).  
  - HTTP/API error mapping (status codes, error payload formats).  
  - Logging strategy (central logger, error middleware).  

- Characterize patterns:
  - Where errors originate (domain services vs. controllers vs. infrastructure).  
  - Whether errors are fail‑fast, retried, or compensated (e.g. retries for transient DB/network errors).  
  - How user‑visible messages are generated vs. internal technical details (separation of concerns).

Document:
- Common error categories (auth, validation, business rules, system failures).  
- Typical handling flows and any notable inconsistencies.

### Step 7: Synthesis & Gaps

Connect the dots into a coherent picture of the domain and architecture. [skillmp](https://skillmp.ru/skills/tools/ide-plugins/exploring-codebases-skill-dlya-analiza-kodovoy-bazy/)

- Summarize:
  - High‑level architecture: main modules and how they collaborate to support auth, user/customer management, product catalog, ordering, and inventory.  
  - The order lifecycle as a state machine, tied to schema and code.  
  - How inventory is kept (or not kept) consistent with orders.  
  - The error‑handling philosophy and tracing/auditing capabilities.

- List:
  - **Unknowns / Ambiguities**: unclear transitions, missing audits, inventory edge cases.  
  - **Potential follow‑up questions** developers should ask.  
  - **Overrides applied/ignored** so the reader knows what was emphasized or skipped.

Output everything as a single markdown report, with headings and concise sections.

***

## RULES

**Always:**

- Stay read‑only: do not modify code, configs, or schemas.  
- Prefer lazy, targeted file reads over scanning the entire repo.  
- Make module and state‑machine inferences explicit (describe how you inferred them).  
- Distinguish between what’s enforced in schema (constraints, enums) and what’s enforced in code (business rules). [prisma](https://www.prisma.io/docs/llms-full.txt)
- Call out gaps or uncertainties instead of guessing.

**Never:**

- Assume specific paths like `src/` or `prisma/schema.prisma` must exist; treat them as common defaults, not requirements.  
- Claim that a state machine or inventory mechanism exists when you couldn’t find supporting code or schema.  
- Infer behavior solely from naming without checking representative implementations.  
- Generate or change commits; this skill is descriptive only.

***

## Overrides

Free‑form instructions at the end of the invocation override defaults. Examples:

- Scope: `only orders and inventory`, `focus on auth + users`, `ignore products`.  
- Depth: `quick overview`, `deep dive into state machine`.  
- Output: `bullet list summary`, `include example code references (file:line)`.

Unrecognized or contradictory overrides: defaults win; note them under “Overrides ignored”.

***

If you’d like, I can now help you tailor this spec to your exact environment (e.g. “Cursor skills”, “Claude skills”, or your own orchestration format) and add more concrete examples of how it should search for `prisma/schema.prisma` when present but still remain generic.