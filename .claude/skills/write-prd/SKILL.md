---
name: write-prd  
description: Produces `docs/PRD.md` as a consolidated Product Requirements Document based on ADRs, RFC, FDD, and meeting outputs, including context, scope, requirements, risks, metrics, and acceptance criteria.
---

# Write PRD

Creates a Product Requirements Document at `docs/PRD.md` that consolidates what has already been decided in ADRs, RFC, FDD, and meeting outputs. The PRD answers **what** we are building and **why**, not how to implement it (which lives in the FDD). [product-blueprint](https://product-blueprint.com/prd-product-requirements-document/)

This skill assumes that at least some of the following already exist: ADRs, RFC, FDD, and/or a meeting summary/transcript.

## INPUT

Free-form. The skill infers what was passed. Any combination works:

- Paths to existing documents:
  - `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/`, `docs/summary.md`, `docs/transcription.md`.  
- Natural-language context:
  - Short description of the feature, target users, and business goals.  
- Optional overrides:
  - “Write PRD for feature X only (ignore legacy parts).”  
  - “Append to existing PRD instead of rewriting.”  

If the skill cannot find enough information to infer the feature’s problem, scope, and key requirements from ADR/RFC/FDD/meeting inputs, it should abort with:

> “Not enough information to generate a PRD. Please provide at least one of: RFC, FDD, ADRs, or a meeting summary describing the feature.”

## OUTPUT

A single file (by default `docs/PRD.md`) containing a PRD with at least the following sections: [github](https://github.com/ugur10/prd-template)

- Summary and feature context  
- Problem and motivation  
- Target audience and usage scenarios  
- Objectives and success metrics  
- Scope (in-scope and out-of-scope)  
- Functional requirements  
- Non-functional requirements  
- Key decisions and trade-offs  
- Dependencies  
- Risks and mitigation  
- Acceptance criteria  
- Testing and validation strategy  

The PRD must:

- Be consistent with ADRs, RFC, and FDD (no contradictions).  
- Use product/feature language (user-oriented) while staying grounded in the existing technical docs.  
- Prefer real, documented content over invented details; when the underlying materials are sparse, the PRD should explicitly note gaps instead of fabricating content.

***

## EXECUTION STEPS

### Step 1: Resolve Input and Target Path

Parse the input and extract:

- **Existing documents**:
  - Check for `docs/RFC.md`, `docs/FDD.md`, and ADR files under `docs/adrs/`.  
  - Meeting summary/transcript: `docs/summary.md`, `docs/transcription.md`, or a similar file.  
- **Feature scope hints**:
  - Name or identifier of the feature (e.g., “order status notifications”).  
- **PRD path override**:
  - If specified, use that; otherwise default to `docs/PRD.md`.

If `docs/` does not exist and cannot be created, abort with a clear message.

If an existing `docs/PRD.md` is present:

- Overwrite only if the user indicates that this PRD should supersede the previous one; otherwise ask whether to:
  - Update/extend the existing PRD, or  
  - Create a new PRD file (e.g., `docs/PRD-feature-x.md`).

### Step 2: Load Supporting Context

Load and parse:

- **RFC (`docs/RFC.md`)**:
  - Context, problem, proposal, alternatives, open questions, impact, risks. [productboard](https://www.productboard.com/blog/product-requirements-document-guide/)
- **FDD (`docs/FDD.md`)**:
  - Detailed behavior and flows, public contracts, error handling, resilience, observability.  
- **ADRs (`docs/adrs/ADR-*.md`)**:
  - Decisions and trade-offs that affect the feature. [github](https://github.com/adr/madr)
- **Meeting summary/transcript**:
  - Closed decisions, functional requirements, items explicitly discarded or deferred, risks discussed.

Use these as the **source of truth**. The PRD should not invent new decisions or requirements that are not supported by these inputs.

### Step 3: Extract Core PRD Elements

From the combined context, derive:

#### 3.1 Summary and feature context

- High-level explanation of what this feature is and where it fits in the product.  
- One or two paragraphs intended for stakeholders to quickly understand the feature and its purpose. [figma](https://www.figma.com/resource-library/product-requirements-document/)

#### 3.2 Problem and motivation

- Clear statement of the problem(s) this feature solves and why now.  
- Business and/or operational pain points (e.g., user friction, operational cost, reliability issues). [edana](https://edana.ch/en/2025/08/15/product-requirements-document-prd-complete-guide-templates-and-practical-examples/)

#### 3.3 Target audience and usage scenarios

- Target users/personas (e.g., “merchant admins”, “internal ops”, “integrating third-party systems”).  
- Main usage scenarios:
  - A concise list of user-focused scenarios or user stories that illustrate how the feature will be used. [startupproject](https://startupproject.org/templates/prd/)

#### 3.4 Objectives and success metrics

- One or more product/feature objectives, each stating:
  - What outcome is desired.  
  - A measurable metric and, when possible, a quantitative target.  
  - Example:  
    - “Increase successful webhook delivery rate from 96% to ≥ 99.5% within 3 months of rollout.”  
    - “Reduce mean time to detect failures by 50%.” [atlassian](https://www.atlassian.com/software/confluence/templates/product-requirements)

When the source material does not declare explicit targets, the skill may propose reasonable targets aligned with the problem statement, clearly labeling them as **proposed**.

#### 3.5 Scope (in-scope and out-of-scope)

- **In scope**:
  - Concrete capabilities the feature will deliver (integrations, flows, UI surfaces, APIs).  
- **Out of scope**:
  - Items explicitly discarded or deferred in discussions (e.g., “support for legacy API X in v1”, “UI for advanced filters”).  

“Out of scope” should capture all exclusion decisions that are clearly documented in meetings, RFC, ADRs, or FDD; the skill must not invent out-of-scope items without evidence.

#### 3.6 Functional requirements

- Derive functional requirements from:
  - Meeting summary/transcript.  
  - FDD flows and public contracts.  
  - RFC proposal and acceptance criteria.  

Represent them as a numbered list of clear, testable statements:

- E.g., “FR-01: The system must send a notification whenever an order transitions to status X.”  

The PRD should include all clearly identified functional requirements that were discussed or are clearly implied by the existing documents; it must not fabricate requirements merely to reach an arbitrary count. [worksbuddy](https://worksbuddy.ai/blogs/what-is-included-in-a-product-requirement-document)

#### 3.7 Non-functional requirements

- Capture performance, reliability, security, compliance, UX responsiveness, etc., consistent with RFC/FDD: [cavaro](https://www.cavaro.io/templates/product-requirements-document)
  - Latency bounds, throughput, data retention, availability targets.  
  - Security constraints (auth, encryption, audit trail).  
  - Regulatory constraints if any.  

#### 3.8 Key decisions and trade-offs

- Summarize important decisions and trade-offs from ADRs and RFC: [adr.github](https://adr.github.io/adr-templates/)
  - Chosen architecture patterns (e.g., outbox vs. direct calls, caching strategies, data models).  
  - Important trade-offs (latency vs. reliability, consistency vs. complexity).  
- Cross-reference ADR IDs and relevant RFC sections.

#### 3.9 Dependencies

- Dependencies on:
  - Other features, services, or teams.  
  - External systems, libraries, or infrastructure.  
  - Data migrations or schema changes.  

When possible, indicate which dependencies are **blocking** vs. **non-blocking**.

#### 3.10 Risks and mitigation

- List the most important risks relevant to the feature, each with: [github](https://github.com/storj/roadmap/blob/main/Product%20Requirements%20Document%20Template)
  - **Description**.  
  - **Probability** (e.g., Low/Medium/High or approximate percentage).  
  - **Impact** (Low/Medium/High, and what it affects).  
  - **Mitigation** (what will be done to reduce likelihood or impact).  

The skill must not invent risks out of thin air; when only a small number of risks are discussed in the source material, include those and avoid padding the list.

#### 3.11 Acceptance criteria

- Define clear, testable acceptance criteria that align with functional and non-functional requirements and success metrics: [figma](https://www.figma.com/resource-library/product-requirements-document/)
  - Enough detail that QA/engineering can say “done/not done”.  
  - Often expressed as scenario-level criteria, not just single requirements.

#### 3.12 Testing and validation strategy

- Overall plan to validate that the feature meets requirements and objectives: [cavaro](https://www.cavaro.io/templates/product-requirements-document)
  - Types of testing: unit, integration, end-to-end, load, usability, etc.  
  - Environments and data needed.  
  - How to validate success metrics post-launch (instrumentation, dashboards, experiments, monitoring).

***

## Step 4: Compose `docs/PRD.md`

Use at least this structure:

```markdown
# Product Requirements Document: <Feature Name>

## Summary and Context

<High-level summary of the feature and its context in the product.>

## Problem and Motivation

<What problem we are solving and why it matters now.>

## Target Audience and Usage Scenarios

### Target Audience

- <Persona or user group 1>
- <Persona or user group 2>

### Usage Scenarios

- <Scenario 1>
- <Scenario 2>
- <Scenario 3>
...

## Objectives and Success Metrics

- Objective 1: <description>
  - Metric: <name>, Target: <numeric target and time frame, or clearly labeled as proposed>
- Objective 2 (optional): ...

## Scope

### In Scope

- <Item 1>
- <Item 2>
- ...

### Out of Scope

- <Out-of-scope item 1 (explicitly discarded/deferred)>
- <Out-of-scope item 2 (explicitly discarded/deferred)>
- ...

## Functional Requirements

1. [FR-01] <requirement text>
2. [FR-02] <requirement text>
3. ...
<Include all clearly identified functional requirements from the source material.>

## Non-Functional Requirements

- <NFR-01> <requirement text>
- <NFR-02> <requirement text>
- ...

## Key Decisions and Trade-offs

- <Decision 1> — trade-offs and rationale. (See ADR-00X, RFC section Y.)
- <Decision 2> — ...

## Dependencies

- <Dependency 1>
- <Dependency 2>
- ...

## Risks and Mitigation

- Risk 1:
  - Description: <text>
  - Probability: <Low/Medium/High or %>
  - Impact: <Low/Medium/High + what is impacted>
  - Mitigation: <strategy>

- Risk 2:
  - Description: <text>
  - Probability: <...>
  - Impact: <...>
  - Mitigation: <...>

## Acceptance Criteria

- <Acceptance criterion 1>
- <Acceptance criterion 2>
- ...

## Testing and Validation Strategy

- <Testing approaches (unit, integration, e2e, load, etc.)>
- <How we will validate success metrics and acceptance criteria post-launch>
```

The skill must:

- Ensure all required sections are present.  
- Prefer completeness and clarity grounded in real ADR/RFC/FDD/meeting content over meeting arbitrary numeric thresholds.  
- Explicitly mention when the available source material is sparse (e.g., “Only three functional requirements were explicitly discussed; additional requirements may be discovered during design.”) instead of silently inventing detail.

***

## RULES

**Always:**

- Derive PRD content from ADRs, RFC, FDD, and meeting outputs; do not contradict them. [productboard](https://www.productboard.com/blog/product-requirements-document-guide/)
- Include all required sections and make success metrics, requirements, and risks as concrete and testable as the source material allows.  
- Keep the PRD focused on **what and why**, leaving detailed implementation to FDD.

**Never:**

- Invent requirements, decisions, or metrics without any basis in the source documents; when proposing a metric or target, clearly label it as a proposal aligned with the problem statement.  
- Omit “Out of scope”, “Functional requirements”, “Risks and mitigation”, or “Testing and validation strategy”.  
- Use the PRD to silently override ADR/RFC/FDD decisions; changes must go through those documents.

This version should be reusable across projects; repository-specific constraints (like “this project wants at least 8 functional requirements”) can live in your templates or `CLAUDE.md`, not in the generic skill.