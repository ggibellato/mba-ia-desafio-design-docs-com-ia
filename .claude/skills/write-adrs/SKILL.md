---
name: write-adrs  
description: Generates one or more Architecture Decision Records (ADRs) in MADR style, stored as separate files under `docs/adrs/`, based on the information provided by the user and existing code context.
---

# Write ADRs

Creates a small set of Architecture Decision Records (ADRs) that capture the main architectural decisions implied by the user’s input (meeting summaries, specs, FDDs, etc.) and, when useful, by the existing codebase. ADRs are written in a MADR-style format and saved as separate Markdown files under `docs/adrs/` by default. [adr.github](https://adr.github.io/adr-templates/)

## INPUT

Free-form. The skill infers what was passed. Any combination works:

- A description of decisions to capture:  
  - Natural language summary, meeting summary, or design notes.  
- Paths to supporting files:  
  - `docs/summary.md`, `docs/fdd/*.md`, `transcription.md`, etc.  
- Optional codebase root or modules to inspect:  
  - `.`, `src/`, `backend/`, `services/orders/`, etc.  
- Optional ADR location override:  
  - “write ADRs under `architecture/decisions/` instead of `docs/adrs/`”  
  - “use prefix `000` instead of `ADR-`”  

The skill must:

1. Derive **candidate decisions** from the provided information (and optionally from code).  
2. Decide which ones deserve ADRs (primary decisions vs. secondary).  
3. Map each chosen decision to a separate ADR file (unless the user explicitly asks for a single ADR).

If no clear decisions can be inferred, the skill should abort with:

> “No clear architectural decisions were identified in the provided information. Please provide decisions or design discussions to record.”

## OUTPUT

- One or more ADR files, created or updated under `docs/adrs/` by default, each named:

  - `ADR-NNN-title-in-kebab-case.md`  

  For example: `ADR-001-outbox-in-mysql.md`, `ADR-002-auth-token-lifetime.md`.

- The numbering `NNN` should be:
  - Sequential within `docs/adrs/`.  
  - Zero-padded to three digits (`001`, `002`, `010`, `123`, …).  

- Each ADR must follow a MADR-like structure with, at minimum, these sections: [github](https://github.com/joelparkerhenderson/architecture-decision-record/tree/main/locales/en/templates/decision-record-template-of-the-madr-project)

  - **Title** (top-level heading)  
  - **Status**  
  - **Context and Problem Statement** (or “Context”)  
  - **Decision**  
  - **Considered Alternatives** (at least 1 real or plausible alternative)  
  - **Consequences** (positive and negative, with explicit trade-offs)  

- ADR must explicitly reference existing code artifacts if any are related to the decision, for example:
  - Specific file paths (`src/orders/service.ts`)  
  - Modules or patterns already in use (e.g., `OrderRepository`, “Outbox pattern in module X”).  

Secondary technical decisions (payload format, timeouts, headers, etc.) may either:

- Become additional ADRs (if they are significant), or  
- Be left to FDD or other docs, if the skill judges they are too low-impact to warrant a full ADR.

The skill should favor a small, meaningful set of ADRs rather than many trivial ones. [ozimmer](https://ozimmer.ch/practices/2022/11/22/MADRTemplatePrimer.html)

***

## EXECUTION STEPS

### Step 1: Resolve Input and Target Directory

Parse the entire input as free-form. Extract:

- **Decision sources**  
  - Meeting summaries, explicit decisions, design notes, user text describing decisions.  
- **Supporting files**  
  - Paths to summaries (`summary.md`), FDDs, PRDs, RFCs, or other context.  
- **Codebase roots** (optional)  
  - Directories where relevant code lives, if given.  
- **ADR directory override** (optional)  
  - If the user specifies a different target (e.g., `docs/decisions/`), use it instead of `docs/adrs/`.

Resolution rules:

- Default ADR directory is `docs/adrs/` if not overridden.  
- If the target directory does not exist:
  - Create it if possible.  
  - If creation is not allowed in the environment, abort with a clear message.  

### Step 2: Identify Candidate Decisions

From the provided text and files, identify **architecturally relevant** decisions. Examples: [adr.github](https://adr.github.io/adr-templates/)

- Choice of patterns or mechanisms (e.g., Outbox pattern, Saga vs. simple transactions).  
- Technology or framework choices (e.g., “use Prisma with PostgreSQL”).  
- Cross-cutting concerns (auth, logging, error handling strategies).  
- Data modeling or integration approaches (e.g., “orders reference customers by ID only”).  

For each candidate, capture:

- Short working title.  
- Context and problem it addresses.  
- At least one plausible alternative (even if it was not chosen).  
- Observed consequences or expected trade-offs.

If no sufficiently clear decisions are found, abort as described above.

### Step 3: Decide Which ADRs to Write

From the candidate decisions, choose which ones become ADRs:

- **Must include**:
  - Main decisions clearly present in the user’s information.  
  - At least one decision that explicitly touches existing code (integration, refactor, pattern adoption, etc.).  

- **May include** (if significant enough):
  - Secondary technical decisions such as:
    - Payload formats.  
    - Timeouts and retries.  
    - Headers and protocol details.  
  - These can be grouped (e.g., a single ADR for “HTTP API conventions”) instead of one ADR per micro-detail.

The skill should:

- Prefer fewer, well-scoped ADRs over many tiny ones.  
- Ensure coverage of the “main decisions discussed in the provided information”.

### Step 4: Determine File Names and Numbers

Scan the target ADR directory to determine the next available ADR number:

- Look for existing ADR files matching `ADR-NNN-*.md`.  
- If none exist:
  - Start at `ADR-001-...`.  
- Otherwise:
  - Use the highest existing `NNN` and increment by 1.  

For each new ADR:

- Convert the title to kebab-case for the filename:
  - Remove accents and punctuation.  
  - Lowercase everything.  
  - Replace spaces with `-`.  
- Generate `ADR-NNN-title-in-kebab-case.md` using the next number.

If the user provided an explicit filename or numbering scheme, follow it for that ADR, but still log the chosen name in the content (e.g., “This is ADR-007 …”).

### Step 5: Gather Code References

Inspect the codebase (if a root path was provided or is discoverable) to find relevant modules or files that implement or are affected by the decision: [scribd](https://www.scribd.com/document/683671003/Architecture-Decision-Records-in-MD-and-GIT)

- Search for:
  - Classes, modules, functions, or patterns related to the decision.  
  - Existing infrastructure components (e.g., message buses, repositories, adapters).  
- Add explicit references to those artifacts in the ADR’s **Context** or **Decision** sections, for example:
  - “This decision affects `src/orders/order-service.ts` and `src/inventory/stock-reserver.ts`.”  
  - “The Outbox pattern is implemented in `src/infra/outbox-publisher.ts`.”

If no codebase root is given or code cannot be read:

- Satisfy the requirement by referencing modules or patterns named in the user input (e.g., “existing Orders module”).

### Step 6: Write Each ADR (MADR-like)

For each selected decision, write an ADR in a MADR-like format with the required sections. [adr.github](https://adr.github.io/madr/)

Minimum structure per ADR:

```markdown
# <Short title of the decision>

- Status: <proposed | accepted | rejected | deprecated | superseded by ADR-NNN>
- Date: <YYYY-MM-DD>
- Deciders: <optional, if known>
- Related: <optional links to PRD, RFC, FDD, other ADRs>

## Context and Problem Statement

<Describe the context, forces, and problem to be solved in a few sentences.>

## Decision

<Describe the chosen option clearly and unambiguously.>

## Considered Alternatives

- <Alternative 1> — brief description of what it is.
- <Alternative 2> — optional, if it was discussed or is a plausible option.
- ...

## Consequences

### Positive

- <Positive consequence 1>
- <Positive consequence 2>
- ...

### Negative

- <Negative consequence 1>
- <Negative consequence 2>
- ...

```

Requirements:

- **Status** must be set (typically `proposed` or `accepted`, depending on the context you infer).  
- **Considered Alternatives** must include at least one real or plausible alternative (no fake or trivial placeholders).  
- **Consequences** must mention both positive and negative aspects, making the trade-offs explicit (e.g., performance vs. complexity). [ozimmer](https://ozimmer.ch/practices/2022/11/22/MADRTemplatePrimer.html)
- Where relevant, reference:
  - Code files or modules.  
  - Related documents (PRD, RFC, FDD, other ADRs).  

### Step 7: Save ADR Files

For each ADR:

- Write the composed content to the chosen filename under the target directory (`docs/adrs/` by default).  
- Do not overwrite existing ADRs with the same number and title unless explicitly instructed by the user to “update ADR-XXX”.

If an ADR number conflict or filename conflict is detected:

- Increment the number until a free slot is found, or  
- If instructed to update, modify the existing ADR instead, preserving history semantics (e.g., marking previous ADR as `superseded` and referencing the new one, if that is the intention). [github](https://github.com/joelparkerhenderson/architecture-decision-record/tree/main/locales/en/templates/decision-record-template-of-the-madr-project)

***

## RULES

**Always:**

- Use separate files for each ADR (one decision per ADR).  
- Use the `ADR-NNN-title-in-kebab-case.md` pattern under `docs/adrs/` by default.  
- Include at least the sections: Status, Context and Problem Statement, Decision, Considered Alternatives, Consequences.  
- Ensure at least one ADR explicitly references existing code artifacts (files, modules, or patterns).  
- Make trade-offs explicit in the Consequences section (both positive and negative). [adr.github](https://adr.github.io/adr-templates/)

**Never:**

- Invent decisions that are not supported by the provided information or obvious project context.  
- Omit “Considered Alternatives” or “Consequences”.  
- Hide trade-offs by listing only positives or only negatives.  
- Pack multiple independent decisions into a single ADR (unless the user explicitly asks for that).  
- Break the numbering or naming convention unless the user deliberately overrides it.
