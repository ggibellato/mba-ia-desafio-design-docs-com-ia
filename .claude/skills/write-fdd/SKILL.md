---
name: write-fdd  
description: Produces `docs/FDD.md` with an implementation-oriented Feature Design Document, detailed enough for developers to start coding, including flows, contracts, errors, resilience, observability, and integration with existing systems.
---

# Write FDD

Creates a detailed Feature Design Document at `docs/FDD.md` describing **how to implement** the feature. The FDD is the most technical document in the set: it must be actionable enough that a developer can read it and start coding. It is derived from higher-level documents (PRD, RFC, ADRs, meeting summaries) but focuses on concrete flows, contracts, error handling, resilience, observability, dependencies, and integration with existing code.

## INPUT

Free-form. The skill infers what was passed. Any combination works:

- Problem and feature context:
  - Natural-language description, PRD, RFC, or meeting summary that defines what this feature must do.
- Design decisions:
  - ADRs (`docs/adrs/ADR-*.md`), RFCs (`docs/RFC.md`), notes about chosen patterns (outbox, DLQ, retries, etc.).
- Codebase and module paths:
  - Project root or relevant modules, e.g. `.`, `src/`, `services/webhooks/`, `backend/orders/`, etc.
- Environment:
  - Information about existing infrastructure (queues, databases, HTTP APIs).

Optional overrides:

- Different FDD path:
  - “Write to `docs/fdd/webhooks.md` instead of `docs/FDD.md`.”
- Project type:
  - “This is a greenfield project” (no existing system), or
  - “This is an existing system; integrate with modules X/Y/Z.”

If the skill cannot infer the feature’s behavior and responsibilities from the input, it should abort with:

> “No clear feature behavior or objectives were found in the provided information. Please describe what the feature must do at a high level before generating the FDD.”

## OUTPUT

A single file (by default `docs/FDD.md`) containing a technical, implementation-oriented FDD with at least the following sections: [learn.microsoft](https://learn.microsoft.com/en-us/dynamics365/guidance/patterns/create-functional-technical-design-document)

- Context and technical motivation  
- Technical objectives  
- Scope and exclusions  
- Detailed flows (including outbox creation, worker processing, retries, DLQ)  
- Public contracts (HTTP endpoints with example payloads, headers, status codes, semantics)  
- Error matrix (with error codes in the `WEBHOOK_*` pattern)  
- Resilience strategies (timeouts, retries, backoff, fallback)  
- Observability (metrics, logs, tracing)  
- Dependencies and compatibility  
- Technical acceptance criteria  
- Risks and mitigation  
- If the project is **not** greenfield:  
  - **Integration with existing system** section, with real file paths and module names.

The FDD should be concrete enough that a developer can:

- Understand the end-to-end flow.  
- See exactly which endpoints to implement and what they accept/return.  
- Know how errors are represented and handled.  
- Know which modules, files, and patterns to modify or extend in an existing codebase. [atlassian](https://www.atlassian.com/work-management/knowledge-sharing/documentation/software-design-document)

***

## EXECUTION STEPS

### Step 1: Resolve Input and Target Path

Parse the input and extract:

- **Feature context**:
  - PRD/RFC references, meeting summary, or inline description of the feature.  
- **Design decisions**:
  - ADRs and key architectural choices that constrain implementation.  
- **Codebase root / modules**:
  - Paths where the feature lives or will live.  
- **Project type** (if stated):
  - Greenfield vs. existing system.  

Determine target FDD path:

- Default: `docs/FDD.md`.  
- If an override path is provided, use it instead.  

If `docs/` (or the overridden directory) does not exist and cannot be created, abort with a clear message.

If an existing FDD file is present:

- Update/replace it only if the user indicates that this FDD supersedes or refreshes the previous one; otherwise, ask whether to overwrite or write a new file.

### Step 2: Load Supporting Context

When paths are provided, load:

- **RFC** (`docs/RFC.md`):
  - For architecture-level proposal, constraints, alternatives, and risks.  
- **ADRs** (`docs/adrs/ADR-*.md`):
  - For specific decisions that affect implementation (outbox pattern, error handling, storage choices, etc.). [adr.github](https://adr.github.io/adr-templates/)
- **Meeting summary / transcript summary**:
  - For closed decisions and functional requirements, especially around flows and error scenarios.  
- **Existing code** (non-greenfield):
  - Modules that will be extended or integrated (e.g., `src/orders/order-service.ts`, `src/webhooks/dispatcher.ts`).

Use these as constraints and guides for the FDD; do not repeat RFC content verbatim.

### Step 3: Determine Greenfield vs. Existing System

From the input and code inspection, determine:

- **Greenfield**:
  - No existing implementation; paths may not exist yet.  
  - The FDD defines new modules and files to create.  
- **Existing system**:
  - There are modules, services, or flows already in place.  
  - The feature extends, modifies, or integrates with them.

If it is **not** clearly greenfield, treat it as an existing system and:

- Require an **“Integration with existing system”** section.  
- Resolve real file paths and modules to reference in that section.

### Step 4: Extract FDD Core Content

From context (PRD, RFC, ADRs, meetings) and code (if present), derive:

4.1 **Context and technical motivation**

- Why this feature exists from a technical point of view:
  - Technical problems being solved (e.g., unreliable webhook delivery).  
  - Constraints (throughput, latency, consistency, existing interfaces).  

4.2 **Technical objectives**

- Clear, technical goals:
  - Reliability objectives (e.g., at-least-once delivery, idempotency).  
  - Performance and scalability targets (where known).  
  - Operability and maintainability goals.

4.3 **Scope and exclusions**

- In-scope:
  - Specific behaviors, surfaces, and components to be implemented.  
- Out-of-scope:
  - Related but explicitly excluded features or use cases.

4.4 **Detailed flows**

Especially for webhook/outbox-like systems, describe end-to-end flows in detail: [youtube](https://www.youtube.com/shorts/vdx4-z5CoBk)

- **Outbox event creation**:
  - When and where events are created (e.g., inside transactional boundaries).  
  - Data captured in outbox records.  
- **Worker processing**:
  - How workers load events, send webhooks, and mark success/failure.  
  - Ordering, concurrency, and batching behavior.  
- **Retry logic**:
  - Retry schedule, max attempts, idempotency considerations.  
- **Dead-letter queue (DLQ)**:
  - When events are moved to DLQ.  
  - How DLQ entries are inspected and remediated.

These flows should be detailed enough to inform implementation and tests (pseudocode, sequence steps, or sequence diagrams are acceptable).

4.5 **Public contracts**

Define the external interfaces:

- **HTTP endpoints**:
  - Paths, methods, query parameters.  
  - Request payloads with concrete examples (JSON bodies, headers).  
  - Response payloads, status codes, and semantics (e.g., what 200 vs 202 vs 4xx vs 5xx mean).  
- **Webhook payloads** (if relevant):
  - Event structures, required/optional fields.  
  - Headers and signature/auth mechanisms. [mgsoftware](https://www.mgsoftware.nl/en/templates/technical-specification-template)

4.6 **Error matrix (`WEBHOOK_*` codes)**

Define a matrix of expected errors:

- Each row should contain:
  - Error code (e.g., `WEBHOOK_TIMEOUT`, `WEBHOOK_INVALID_PAYLOAD`, `WEBHOOK_DESTINATION_4XX`).  
  - HTTP status (if applicable).  
  - Description and cause.  
  - Where it is raised (which component/endpoint).  
  - How it should be handled or surfaced.

Ensure codes follow the `WEBHOOK_*` naming pattern consistently.

4.7 **Resilience strategies**

Describe how the system remains robust: [solarwinds](https://www.solarwinds.com/blog/sustaining-digital-resilience-with-secure-by-design)

- **Timeouts**:
  - Default timeouts for outbound calls, and justification.  
- **Retries and backoff**:
  - Retry policy (fixed, exponential backoff), max attempts, jitter.  
- **Fallbacks**:
  - Alternative paths, degraded modes, or manual intervention strategies.  
- **Circuit breakers / rate limits** (if applicable):
  - How to protect downstream systems and your own infrastructure.

4.8 **Observability**

Specify how the feature will be observed in production: [dev](https://dev.to/adeolu102/engineering-design-document-reusable-observability-platform-v2-54gb)

- **Metrics**:
  - Key counters and gauges (e.g., delivered events, failed events, DLQ entries, latency).  
- **Logs**:
  - Log structure, log levels for different conditions, correlation IDs.  
- **Tracing**:
  - Spans for key operations (event creation, dispatch, retry, DLQ move).  
  - Propagation of trace context through HTTP calls.

4.9 **Dependencies and compatibility**

Enumerate:

- External services, databases, queues, libraries.  
- Version compatibility and migration concerns.  
- Any impact on existing consumers (e.g., compatible changes vs. breaking changes).

4.10 **Technical acceptance criteria**

Define concrete, testable criteria:

- What must be true for the implementation to be considered “done”:
  - Functional acceptance (scenarios that must pass).  
  - Non-functional criteria (latency thresholds, error budgets, throughput).  
  - Testing requirements (unit, integration, end-to-end tests, load tests).

4.11 **Risks and mitigation**

List:

- Implementation risks (complexity, unknowns).  
- Operational risks (incident profiles, failure modes).  
- Proposed mitigation strategies (phased rollout, feature flags, kill switches, fallback behavior).

### Step 5: Integration with Existing System (Non-Greenfield)

If the project is **not** greenfield, the FDD **must** include a section:

`## Integration with existing system`

This section must:

- Name and reference **real file paths and modules** in the codebase.  
- Describe how the new module/feature will integrate with each relevant artifact, for example:
  - How a `changeStatus` method is extended or modified.  
  - How existing error classes or error-handling middleware are reused.  
  - Which repository, service, or controller is hooked into the new flows.  
- If you cannot find relevant files but the project is declared non-greenfield:
  - State the limitation explicitly and ask for clarification, rather than inventing file paths.

This section is mandatory for non-greenfield projects; omit it only when it is clearly a new system.

### Step 6: Write FDD Structure

Compose `docs/FDD.md` with at least this structure:

```markdown
# Feature Design Document: <Feature Name>

## Context and Technical Motivation

<Explain why this feature is needed from a technical perspective and what problems it solves.>

## Technical Objectives

- <Objective 1>
- <Objective 2>
- ...

## Scope and Exclusions

### In Scope

- <Item 1>
- <Item 2>

### Out of Scope

- <Item 1>
- <Item 2>

## Detailed Flows

### Outbox Event Creation

<Step-by-step description or sequence diagram of how and when outbox events are created.>

### Worker Processing

<How workers fetch, process, and mark events; concurrency, ordering, batching.>

### Retry Logic

<Retry policy, idempotency strategy, error handling on retry.>

### Dead-Letter Queue (DLQ)

<Conditions for DLQ, structure of DLQ entries, remediation processes.>

## Public Contracts

### HTTP Endpoints

- `POST /...`  
  - Request body (example)  
  - Required headers  
  - Response status codes and payloads  
  - Semantics

### Webhook Payloads (if applicable)

<Describe payload format, headers, authentication/signature, and examples.>

## Error Matrix (`WEBHOOK_*`)

| Code               | HTTP Status | Description                      | Where Raised                  | Handling Strategy        |
|--------------------|------------|----------------------------------|-------------------------------|--------------------------|
| `WEBHOOK_TIMEOUT`  | 504        | ...                              | <component or endpoint>       | <retry / DLQ / log etc.> |
| `WEBHOOK_INVALID_PAYLOAD` | 400 | ...                              | <component or endpoint>       | ...                      |

## Resilience Strategies

- Timeouts: <values and rationale>
- Retries and backoff: <policy details>
- Fallbacks: <if any>
- Other resilience mechanisms: <circuit breakers, rate limits, etc.>

## Observability

### Metrics

- <Metric name> — <what it measures and why>

### Logs

- <Log categories, levels, structure, correlation IDs>

### Tracing

- <Key spans and propagation strategy>

## Dependencies and Compatibility

- <External services, queues, databases, libraries>
- <Compatibility considerations and migrations>

## Integration with Existing System

<For non-greenfield projects: list concrete file paths and modules, describe how the feature integrates with each (methods extended, errors reused, hooks added).>

## Technical Acceptance Criteria

- <Criterion 1>
- <Criterion 2>
- ...

## Risks and Mitigation

- <Risk 1> — <mitigation>
- <Risk 2> — <mitigation>
```

You can optionally include diagrams (system, sequence, state, data layout) in the Detailed Flows or Observability sections when they clarify complex behavior, but they are not required.

### Step 7: Save FDD

Write the composed FDD to the target file (default `docs/FDD.md`):

- Overwrite existing content only when:
  - The user indicates this FDD supersedes the previous one, or  
  - The existing file is clearly a placeholder/template.

Otherwise, ask whether to replace or version it.

***

## RULES

**Always:**

- Produce an implementation-oriented FDD detailed enough for a developer to start coding.  
- Include, at minimum: Context and technical motivation; Technical objectives; Scope and exclusions; Detailed flows; Public contracts; Error matrix with `WEBHOOK_*` codes; Resilience strategies; Observability; Dependencies and compatibility; Technical acceptance criteria; Risks and mitigation. [linkedin](https://www.linkedin.com/pulse/anatomy-effective-technical-design-document-rashedul-islam-1vsuc)
- For non-greenfield projects, include **“Integration with existing system”** with real file paths and modules.  

**Never:**

- Leave flows vague or purely conceptual; they must be specific and actionable.  
- Omit the error matrix or use inconsistent error code naming.  
- Invent code paths or modules that do not exist (for existing systems); if unsure, state the limitation.  
- Duplicate the RFC’s high-level rationale without adding concrete implementation detail.

If you’d like, we can next wire this skill explicitly to consume your meeting summary and RFC outputs so that the three documents line up cleanly (summary → RFC → FDD + ADRs).