---
name: write-tracker  
description: Produces and updates `docs/TRACKER.md`, a Markdown table that cross-references items in PRD/RFC/FDD/ADRs to their origin in transcripts or code, reducing AI hallucinations and ensuring documentation integrity.
---

# Write Tracker

Creates and maintains `docs/TRACKER.md`, a Markdown table that maps each documented item (requirements, decisions, constraints, trade-offs, etc.) back to its original source in either transcripts or code. This tracker acts as a cross-reference matrix, allowing any reader (human or AI) to see where each documented statement came from and helping prevent hallucinations by tying documentation to verifiable sources. [taylorandfrancis](https://taylorandfrancis.com/knowledge/Engineering_and_technology/Computer_science/Traceability_matrix/)

The tracker must be **updated every time** another skill generates or updates a document (PRD, RFC, FDD, ADRs, etc.).

## INPUT

Free-form. The skill infers what was passed. Typical inputs:

- Documents that have just been created or updated:
  - Paths such as `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-002-outbox-no-mysql.md`.  
- Optional references to source materials:
  - Transcript or summary path (e.g., `docs/transcription.md`, `docs/summary.md`).  
  - Code paths or module roots (e.g., `src/`, `src/modules/orders/`).  
- Optional instructions:
  - “Update tracker only for new ADRs.”  
  - “Re-scan RFC and FDD for new items.”  
  - “Do a full rebuild of TRACKER.md.”

If the skill is invoked without any indication of which document(s) changed and cannot infer that from context, it should:

> “Cannot determine which documents were added or modified. Please specify at least one document path (e.g., `docs/RFC.md` or `docs/FDD.md`) to update the tracker.”

## OUTPUT

A single file `docs/TRACKER.md` containing **one Markdown table** with the following exact header:

```markdown
| ID | Document | Type | Content (summary) | Source | Location |
| --- | --- | --- | --- | --- | --- |
| | | | | | |
```

Each subsequent row represents a **single documented item** and must follow:

- **ID**: unique identifier for the item, following patterns like:
  - `PRD-FR-01` (functional requirement from PRD)  
  - `PRD-NFR-02` (non-functional requirement)  
  - `RFC-ALT-02` (alternative in RFC)  
  - `RFC-Q-01` (open question in RFC)  
  - `FDD-CONTRACT-03` (contract in FDD)  
  - `FDD-FLOW-01` (flow in FDD)  
  - `ADR-002` (ADR with number 002)  
- **Document**: the file where the item appears, e.g.:
  - `docs/PRD.md`  
  - `docs/RFC.md`  
  - `docs/FDD.md`  
  - `docs/adrs/ADR-002-outbox-no-mysql.md`  
- **Type**: classification such as:
  - `Functional Requirement`, `Non-Functional Requirement`, `Decision`, `Constraint`, `Trade-off`, `Flow`, `Contract`, `Open Question`, etc.  
- **Content (summary)**: one-line summary of the item’s content.  
- **Source**: either `TRANSCRIPT` or `CODE`.  
- **Location**:
  - For `TRANSCRIPT`: timestamp + speaker, e.g. `[09:17] Diego`.  
  - For `CODE`: concrete file path, e.g. `src/modules/orders/order.service.ts`.

Coverage requirement:

- At least **80%** of identifiable items across your documents must have a corresponding row in the tracker.  
- The skill should aim higher when feasible, but must **not** fabricate sources just to increase coverage. [jamasoftware](https://www.jamasoftware.com/requirements-management-guide/requirements-traceability/traceability-matrix/)

The skill **must preserve** existing rows when updating, adding new rows for new items and updating rows only when the underlying item has clearly changed.

***

## EXECUTION STEPS

### Step 1: Resolve Inputs and Ensure `docs/TRACKER.md` Exists

- Determine:
  - Which documents have been created or updated (from explicit paths or recent context).  
  - Available sources:
    - Transcript(s): `docs/transcription.md`, `docs/summary.md`, or others.  
    - Code roots: `src/`, `backend/`, `services/`, etc.

- Ensure `docs/` exists:
  - If it does not and cannot be created, abort with a clear message.

- If `docs/TRACKER.md` does **not** exist:
  - Create it with a brief title and the mandatory header:

    ```markdown
    # Documentation Tracker

    | ID | Document | Type | Content (summary) | Source | Location |
    | --- | --- | --- | --- | --- | --- |
    | | | | | | |
    ```

### Step 2: Load Current Tracker and Documents

- Load existing `docs/TRACKER.md` and parse all current rows (skip the header).  
- Load each relevant document that has just been created/updated:
  - `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/*.md`, and any other document provided.  
- Optionally load transcript and/or transcript summary to map items back to timestamps/speakers.  
- Optionally use code paths to map items to concrete file locations when `Source = CODE`.

### Step 3: Identify Items in Documents

For each loaded document, extract **identifiable items** that should appear in the tracker: [jamasoftware](https://www.jamasoftware.com/requirements-management-guide/requirements-traceability/traceability-matrix/)

- **PRD**:
  - Functional requirements.  
  - Non-functional requirements.  
  - Constraints.

- **RFC**:
  - Decisions.  
  - Alternatives considered.  
  - Open questions.  
  - Trade-offs and constraints.  

- **FDD**:
  - Detailed flows (e.g., outbox creation, worker processing, retry, DLQ).  
  - Public contracts (HTTP endpoints, payloads, headers, status codes).  
  - Error codes (e.g., `WEBHOOK_*`).  
  - Resilience patterns (timeouts, retries, backoff, fallback).  
  - Observability items (key metrics, logs, tracing).  
  - Dependencies and compatibility notes.  
  - Technical acceptance criteria.  
  - Risks and mitigations.

- **ADRs**:
  - Each ADR yields at least:
    - One `Decision`.  
    - One or more `Trade-off` items (from the Consequences section). [github](https://github.com/adr/madr)

For each item, derive:

- A stable **ID**:
  - If the document already encodes IDs (e.g., `FR-01`), reuse them with a prefix like `PRD-FR-01`.  
  - Otherwise, generate IDs consistently, for example:
    - PRD functional: `PRD-FR-01`, `PRD-FR-02`, …  
    - PRD non-functional: `PRD-NFR-01`, …  
    - RFC alternatives: `RFC-ALT-01`, `RFC-ALT-02`, …  
    - RFC open questions: `RFC-Q-01`, …  
    - FDD contracts: `FDD-CONTRACT-01`, …  
    - FDD flows: `FDD-FLOW-01`, …  
    - FDD errors: `FDD-ERROR-01`, …  
    - ADRs: use ADR number, e.g. `ADR-002`.  
- **Type** based on the section and content.  
- One-line **Content (summary)** describing the essence of the item.

### Step 4: Determine Source and Location

For each item:

- If it is clearly derived from meeting/transcript discussion:
  - Set **Source** = `TRANSCRIPT`.  
  - For **Location**:
    - Prefer timestamp + speaker, if the transcript or summary provides that (e.g., `[09:17] Diego`).  
    - If only a top-level summary exists (no timestamps), use a best-effort description such as:
      - `Transcript summary: "Closed decisions" section`  
      - But do not fabricate precise timestamps.

- If it is clearly tied to existing code:
  - Set **Source** = `CODE`.  
  - For **Location**:
    - Use precise file paths (e.g., `src/modules/orders/order.service.ts`).  
    - Optionally include function/method names when helpful (e.g., `src/modules/orders/order.service.ts#changeStatus`).  

- If the origin is ambiguous (it seems inferred, not clearly from transcript or code):
  - Prefer to:
    - Leave the item out of the tracker, or  
    - Map it conservatively based on the best available hint and mention in the summary that the mapping is approximate.  
  - Do **not** invent transcript timestamps or fake code references just to fill the table.

### Step 5: Merge with Existing Tracker

- For each newly identified item, check if there is already a row in `TRACKER.md` with the same **ID**:
  - If yes:
    - Update the row only if the underlying item has clearly changed (e.g., updated summary or new source location).  
  - If no:
    - Append a new row to the table with fields:
      - `ID`, `Document`, `Type`, `Content (summary)`, `Source`, `Location`.

- Preserve existing rows that still correspond to valid items in the documents.  
- Optionally, mark rows whose underlying items have been removed from documents, but do not silently delete them unless you explicitly choose a pruning behavior.

### Step 6: Check Coverage (≥ 80%)

After merging:

- Compute coverage:

  - `coverage = (number of identifiable items present in TRACKER rows) / (total identifiable items across PRD/RFC/FDD/ADRs)`

- If coverage is below **80%**:
  - Attempt to add more obvious items until the 80% threshold is reached, without inventing sources.  
  - If coverage still cannot reach 80% due to missing or ambiguous sources:
    - Write the best possible tracker and optionally add a note at the top of `TRACKER.md`, such as:
      - `Current estimated coverage: 65%. Some items could not be safely mapped to a clear source (transcript or code).`

Under no circumstance should the skill fabricate transcript timestamps or code locations to artificially raise coverage. [perforce](https://www.perforce.com/blog/alm/how-create-traceability-matrix)

### Step 7: Save `docs/TRACKER.md`

- Regenerate the entire table content (header + rows), sorted if desired:
  - For example, sort by `Document` then `ID`, or keep insertion order.  
- Overwrite `docs/TRACKER.md` with the updated table.  
- Ensure the file remains valid Markdown with exactly one table using the defined header.

***

## RULES

**Always:**

- Maintain `docs/TRACKER.md` as a **single Markdown table** with the exact header:

  ```markdown
  | ID | Document | Type | Content (summary) | Source | Location |
  | --- | --- | --- | --- | --- | --- |
  | | | | | | |
  ```  

- Update the tracker whenever PRD, RFC, FDD, or ADR documents are created or modified.  
- Use stable, meaningful IDs (e.g., `PRD-FR-01`, `RFC-ALT-02`, `FDD-CONTRACT-03`, `ADR-002`).  
- Set `Source` to either `TRANSCRIPT` or `CODE` and provide a concrete `Location` (timestamp+speaker or file path).  
- Aim for at least **80%** coverage of identifiable items across documents. [taylorandfrancis](https://taylorandfrancis.com/knowledge/Engineering_and_technology/Computer_science/Traceability_matrix/)

**Never:**

- Invent transcript timestamps, speakers, or code paths that do not exist.  
- Claim 80% coverage by adding fictitious entries.  
- Remove valid existing tracker rows silently when items still exist in their documents.  
- Use `TRACKER.md` for free-form narrative; it must remain a structured cross-reference matrix.

This keeps the tracker concept intact but fully Anglicizes the field names and codes so all your skills can reason about it consistently while still enforcing traceability.