---
name: write-rfc  
description: Produces `docs/RFC.md` with an architecture-level technical proposal, summarizing the chosen approach, alternatives, open questions, impact, risks, and links to related ADRs, for team review.
---

# Write RFC

Creates a concise architecture-level Request for Comments document based on the user’s input (meeting summaries, specs, FDDs, ADRs, etc.). By default, the RFC is written to `docs/RFC.md` under the `docs/` folder, unless the user explicitly requests a different path.

## INPUT

Free-form. The skill infers what was passed. Any combination works:

- Problem and context description:
  - Natural-language description of the problem, constraints, and goals.
- Design information:
  - Meeting summaries, transcript summaries, ADRs, FDDs, PRDs, or notes describing the proposed solution and its trade-offs.
- Participants:
  - Names of meeting attendees or stakeholders (used as reviewers).
- Codebase or document paths:
  - `docs/adrs/`, `docs/FDD.md`, `docs/PRD.md`, `docs/summary.md`, etc.

Optional overrides:

- Different RFC path:
  - “Write to `docs/rfcs/new-solution.md` instead of `docs/RFC.md`.”
- Status or lifecycle:
  - “Status should start as Draft / In Review / Accepted / Rejected.”
- Diagrams:
  - “Include diagrams if useful”, “Skip diagrams”, “Prefer sequence diagrams for the protocol part”, etc. 

If the skill cannot infer a clear problem or proposed solution from the input, it should abort with:

> “No clear problem statement or proposed solution was found in the provided information. Please describe the problem and the intended solution at a high level.”

## OUTPUT

A single file (by default `docs/RFC.md`) containing a concise RFC (roughly 2 to 4 pages of content when rendered) with at least the following sections: [lambrospetrou](https://www.lambrospetrou.com/articles/rfc-template/)

- Metadata (author, status, date, reviewers)  
- Executive summary (TL;DR)  
- Context and problem  
- Technical proposal (architecture-level view, not low-level implementation)  
- Alternatives considered  
- Open questions  
- Impact and risks  
- Related decisions (links to ADRs)

Optionally, the RFC may include a Diagrams subsection (either under “Technical Proposal” or as a dedicated section) when visualizations add clarity:

- High-level system architecture / system-context diagram (components and relationships).
- Sequence diagrams (for protocols, APIs, or flows).
- State machine diagrams (for session / lifecycle management).
- Data structure or packet layout summaries (for critical payloads or data contracts).

The RFC must:

- Operate at **architecture level**:
  - Describe what is being proposed and why, not detailed “how to implement” (that goes to the FDD).  
- Reference ADRs:
  - Include links or references to relevant ADR files (e.g., `docs/adrs/ADR-001-outbox-in-mysql.md`).  
- Be written in a clear, professional tone, avoiding unnecessary duplication of FDD content.

***

## EXECUTION STEPS

### Step 1: Resolve Input and Target Path

Parse the entire input as free-form. Extract:

- **Problem/context sources**:
  - Direct user text describing the problem and constraints.
  - References to PRD, FDD, meeting summaries, or transcripts.
- **Proposed solution hints**:
  - Descriptions of the chosen approach, architecture diagrams, patterns, or “we decided to…” statements.
- **Participants / attendees**:
  - Names, emails, or handles that can be used as reviewers.
- **RFC path override** (if any):
  - If the user specifies a different path, use it; otherwise default to `docs/RFC.md`.

If `docs/` does not exist and cannot be created, abort with a clear message.

If there is an existing `docs/RFC.md`:

- Either:
  - Update it (if the user asked to “refresh” or “rewrite” the RFC), or  
  - Abort and ask whether to overwrite or create a separate file.

### Step 2: Load Supporting Context

When paths are provided, load relevant supporting files to enrich the RFC:

- **ADRs** (`docs/adrs/ADR-*.md`):
  - Used to fill in “Related decisions” and inform the proposal, alternatives, and consequences. [adr.github](https://adr.github.io/adr-templates/)
- **FDD** (e.g., `docs/FDD.md`):
  - Used only for high-level understanding of the solution; avoid copying low-level detail.  
- **Meeting summary / transcript summary**:
  - Used to detect decisions, alternatives, and open questions.  
- **PRD / requirements docs**:
  - Used to clarify context, problem, and goals.

If no files are provided, the skill works purely from the natural-language input.

### Step 3: Extract Core RFC Elements

From the combined inputs, identify:

- **Context and problem**:
  - What is wrong with the current state, or what opportunity we want to address.  
  - Constraints (technical, organizational, regulatory, etc.).  

- **Proposed architecture / solution**:
  - High-level structure and approach:
    - Key components and how they interact.  
    - Patterns or technologies being introduced or changed (e.g., message bus, outbox, caching strategy).  
  - Enough detail for reviewers to understand the proposal, not enough to serve as implementation spec. [pandev-metrics](https://pandev-metrics.com/docs/blog/rfc-process-engineering-teams)

- **Alternatives considered**:
  - At least 2–3 viable options whenever possible:
    - Current state (“do nothing”), other patterns, other technologies, different integration strategies.  
  - For each:
    - Short description.  
    - Why it was not chosen (drawbacks, risks, misalignment with constraints).  

- **Open questions**:
  - Explicit questions that came up in meetings.  
  - Gaps in knowledge (e.g., performance assumptions, vendor limitations).  
  - Items to be clarified during review. [github](https://github.com/thecodedrift/rfc/blob/master/0000-template.md)

- **Impact and risks**:
  - Technical impact (architecture, data, performance, reliability).  
  - Operational/process impact (teams, deployments, observability).  
  - Risks and failure modes, plus mitigation ideas (at a high level). [betterprogramming](https://betterprogramming.pub/goals-and-failure-modes-for-rfcs-and-technical-design-documents-c4ee1d1da6ff)

- **Related ADRs**:
  - ADRs that embody decisions this RFC depends on, refines, or proposes.

If any of these elements are weak or missing, the skill should:

- Make reasonable inferences from the provided context, and  
- Explicitly call out uncertainties in the relevant sections instead of pretending they are resolved.

In addition, when diagrams are allowed and likely helpful, infer which diagram types best represent the proposal:

- **High-level system architecture**:
  - When the RFC proposes a new service/system or major changes in component boundaries.

- **Sequence diagrams**:
  - When the RFC defines or changes protocols, API interactions, or cross-service flows.

- **State machine diagrams**:
  - When the RFC introduces or changes lifecycle/state logic (sessions, orders, connections, etc.).

- **Data / packet layouts**:
  - When understanding the structure of key payloads is critical for correctness or compatibility.

If the design is simple enough that diagrams add little value, the skill may skip them.

### Step 4: Build RFC Metadata

Generate a metadata block at the top of the RFC, with at least:

- **Title**:
  - Short, descriptive (e.g., “RFC: Order Processing Architecture with Outbox Pattern”).  
- **Author**:
  - If the user’s name is known, use it; otherwise “TBD”.  
- **Status**:
  - One of: `Draft`, `In Review`, `Accepted`, `Rejected`, `Superseded`.  
  - Default: `Draft` or `In Review`, depending on context.  
- **Date**:
  - Current date in `YYYY-MM-DD` format.  
- **Reviewers**:
  - Use meeting participants when provided; otherwise a placeholder list (“TBD”). [cavaro](https://www.cavaro.io/templates/request-for-comments-rfc)

This can be a Markdown list or a small table.

### Step 5: Write RFC Sections

Compose `docs/RFC.md` using the following minimum structure:

```markdown
# <RFC title>

## Metadata

- Author: <name>
- Status: <Draft | In Review | Accepted | Rejected | Superseded>
- Date: <YYYY-MM-DD>
- Reviewers: <list of names or emails>

## Executive Summary (TL;DR)

<One or two paragraphs summarizing the proposal, key benefits, and the main trade-offs. This section should be understandable on its own.>

## Context and Problem

<Describe the current situation, drivers, constraints, and why change is needed. Include business and technical context as relevant.>

## Technical Proposal

<High-level description of the proposed architecture or solution. Describe main components, data flows, and interactions without going into implementation details that belong in the FDD.>

## Alternatives Considered

- <Alternative 1> — short description and why it was not chosen.
- <Alternative 2> — short description and reasons.
- <Alternative 3> — optional, if relevant.

### Diagrams (optional)

<If useful, include brief, high-level diagrams here or link to them: for example,
- System-context / high-level architecture diagram showing components and relationships.
- Sequence diagram for key API or protocol flows.
- State machine for important lifecycle or session logic.
- Simplified data structure or packet layout for critical payloads.>

<Add a short rationale summarizing why the chosen proposal is preferred over these alternatives.>

## Open Questions

- <Question 1>
- <Question 2>
- <Items that require input from reviewers or further investigation.>

## Impact and Risks

### Impact

- <Technical impact: architecture, data model, performance, reliability.>
- <Operational impact: teams, processes, deployments, monitoring.>

### Risks

- <Risk 1 and potential mitigation.>
- <Risk 2 and potential mitigation.>

## Related Decisions (ADRs)

- [ADR-001-...](docs/adrs/ADR-001-....md) — <short description>
- [ADR-00X-...](docs/adrs/ADR-00X-....md) — <short description>
```

Guidelines:

- Ensure the **Technical Proposal** remains at architecture level and does not duplicate FDD-level implementation details.  
- Use the **Alternatives Considered** section to demonstrate due diligence and trade-offs. [github](https://github.com/thecodedrift/rfc/blob/master/0000-template.md)
- Use **Open Questions** to explicitly invite feedback and highlight areas needing input.  

Notes about diagrams:

- Diagrams are optional, but recommended when:
  - The architecture involves multiple components or services.
  - Flows are non-trivial (e.g., multi-step protocols, auth handshakes).
  - State transitions are central to the design.
- Diagrams should not become low-level implementation blueprints; they must stay at RFC/architecture granularity.

### Step 6: Enforce “No FDD Duplication”

Before finalizing, check the proposal text against any FDD content (if available):

- If the RFC starts to mirror detailed implementation steps, data structures, or low-level APIs:
  - Move those details conceptually into “This will be elaborated in the FDD” phrasing.  
  - Keep only enough detail for architectural review (components, flows, major contracts).  

The RFC must answer “**what we propose and why**”, while the FDD answers “**how to build it in detail**”.

### Step 7: Save RFC

Write the composed content to the target RFC file (default `docs/RFC.md`):

- Overwrite existing content only if:
  - The user explicitly asked to regenerate or update the RFC for this scope; or  
  - The existing RFC is clearly a placeholder/template.  

If overwriting an existing substantive RFC:

- Consider adding or updating a “Status” and “Superseded by RFC-XYZ” note in that previous document, or  
- Ask for confirmation before replacing it, depending on how interactive the environment is.

***

## RULES

**Always:**

- Produce a concise RFC (roughly 2–4 pages worth of content).  
- Include all required sections: Metadata, Executive Summary, Context and Problem, Technical Proposal, Alternatives Considered, Open Questions, Impact and Risks, Related Decisions.  
- Use meeting participants as reviewers when available.  
- Link to ADRs that embody or support the decisions in this RFC.  
- Keep the RFC at architecture level, not implementation spec level. [lambrospetrou](https://www.lambrospetrou.com/articles/rfc-template/)

**Optionally**:

- Include diagrams when they materially improve understanding of:
  - System boundaries and relationships (system-context / high-level architecture diagrams).
  - Cross-service or API flows (sequence diagrams).
  - Lifecycle or session behavior (state machines).
  - Critical data structures or packet formats (layout diagrams).

**Never:**

- Omit “Alternatives Considered”, “Open Questions”, or “Impact and Risks”.  
- Duplicate FDD-level implementation details in the RFC.  
- Invent ADR links that do not exist; if ADRs are missing, either:
  - Reference the intent (e.g., “ADR to be created for X”), or  
  - Recommend creating ADRs as follow-up.  
- Treat the RFC as final approval; its status is part of the metadata and may change as the team reviews it.
