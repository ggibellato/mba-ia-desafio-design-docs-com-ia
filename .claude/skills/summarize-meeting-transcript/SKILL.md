---
name: summarize-meeting-transcript  
description: Reads a meeting transcript and produces a structured summary with decisions, requirements, constraints, integration points, discarded items, deferred items, and secondary technical details.
---

# Summarize Meeting Transcript

Reads a meeting transcript from a `transcription.md` (or similar) file and distills it into a clear, structured summary. The summary is organized into explicit sections for closed decisions, explicit functional requirements, constraints, integration points with existing code, discarded items, items deferred to future phases, and secondary technical details. The transcript file is treated as the primary source of truth whenever there is ambiguity or conflict. [performyard](https://www.performyard.com/articles/how-to-take-meeting-notes)

## INPUT

Free-form. The skill infers what was passed. Any combination works:

- A path to a transcript file:  
  - `transcription.md`, `docs/transcription.md`, `meetings/2026-07-18-transcription.md`, etc.  
- A folder path containing the transcript:  
  - `meetings/`, `docs/sessions/`, etc.  
- Extra natural-language instructions appended anywhere (for example, “focus on backend decisions only”, “ignore HR topics”).

The skill must locate one primary transcript:

1. **Transcript file**  
   - Prefer files whose name clearly indicates a transcript:  
     - `transcription.md`, `transcript.md`, `meeting-notes.md`, `meeting-transcript.md`.  
   - If the input is a file, use it directly.  
   - If the input is a folder, look for one of the above files inside it.  
   - If multiple candidates are found, the skill must either:
     - Choose the one that best matches the input hint (for example, date, topic), or  
     - Abort and list candidates, asking the user to disambiguate. [meetminutes](https://www.meetminutes.in/guide/meeting-templates)

If no transcript file can be found, the skill must abort with a clear error:

> “No transcript file (for example, `transcription.md`) was found under `<path>`. Provide a valid transcript or path.”

## OUTPUT

A single **structured summary** (in English by default, unless the global rules or the user explicitly require another language), with this exact top-level section layout:

1. **Closed decisions**  
2. **Explicit functional requirements**  
3. **Constraints**  
4. **Integration points with existing code**  
5. **Discarded items**  
6. **Items deferred to future phases**  
7. **Secondary technical details**

Each section contains bullet points, written in clear, concise language, and directly grounded in the transcript content. [speaknotes](https://speaknotes.io/blog/meeting-summary-guide)

Additional rules:

- If a section has no content inferred from the transcript, include the section and state explicitly, for example:  
  - “No items identified for this category in the transcript.”  
- The summary must avoid inventing content; where the transcript is vague, the skill should:
  - Either summarize the uncertainty explicitly, or  
  - Mark the item as “ambiguous” or “not fully defined”.

No code or other files are modified. The output is purely descriptive and structured text.

***

## EXECUTION STEPS

### Step 1: Resolve Input

Parse the entire input as free-form. Extract:

- **Transcript reference**  
  - If the input includes a path ending with `.md`, treat it as the candidate transcript.  
  - If the input is a directory, search for transcript-like files (`transcription.md`, `transcript.md`, `meeting-*.md`, etc.). [meetminutes](https://www.meetminutes.in/guide/meeting-templates)
- **Context hints**  
  - Meeting date, topic, project, or other hints embedded in the input.  
- **Extra instructions**  
  - Any remaining text that is not a path or file name; treat as additional guidance for focus or scope (see Step 3).

Resolution rules:

- If exactly one transcript file is resolved, proceed.  
- If multiple plausible transcripts exist:
  - If one matches the hints (e.g., by date or topic), pick that one.  
  - Otherwise, abort and list candidates, asking the user to select one.  
- If no transcript is found, abort with a clear message (see above).

### Step 2: Load Transcript

Read the full content of the resolved transcript file.

- Treat the transcript as the **single source of truth** for:
  - What was decided.  
  - Which requirements, constraints, integrations, and other items were actually mentioned.  
- Do not try to “fix” contradictions between the transcript and other sources; instead:
  - Reflect the transcript faithfully.  
  - Optionally note visible contradictions as part of “Secondary technical details” or within the relevant section. [performyard](https://www.performyard.com/articles/how-to-take-meeting-notes)

If the transcript file is empty or clearly invalid (for example, only a heading with no content), abort with a clear error:

> “The transcript file `<path>` does not contain usable content. Provide a valid meeting transcript.”

### Step 3: Apply Overrides

Interpret extra instructions as natural-language overrides on default behavior. Examples:

- Scope filter:  
  - “focus on backend only”, “ignore UX topics”, “only decisions about API design”.  
- Level of detail:  
  - “high-level summary”, “include more technical details”.  
- Section emphasis:  
  - “emphasize constraints”, “skip secondary technical details section in the output”.

For each recognized override:

- Apply it consistently across all sections.  
- Record internally that the summary is scoped (for example, “Summary focused only on backend decisions, as requested.”).

Overrides **must not**:

- Remove mandatory sections (all seven sections must still be present, though they may state they are empty).  
- Change the language policy enforced by your global `CLAUDE.md` (this spec stays language-agnostic; the global rules decide the final output language).

### Step 4: Extract Core Elements from Transcript

Process the transcript to identify and categorize content into the seven target buckets. Use patterns such as: [monday](https://monday.com/blog/productivity/take-better-meeting-notes/)

- Explicit markers: “we decided”, “decision”, “requirement”, “must”, “constraint”, “out of scope”, “park this for later”, etc.  
- Implicit patterns: clear agreements, commitments, explicit rejection, and postponements.

For each category:

4.1 **Closed decisions**

- Capture decisions that are clearly agreed and not left open.  
- Include:
  - What was decided.  
  - Optionally, a short rationale if present in the transcript.  

4.2 **Explicit functional requirements**

- Capture statements describing **what the system must do**.  
- Prioritize:
  - Concrete, testable behavior (for example, “The system must send a confirmation email after order creation.”). [performyard](https://www.performyard.com/articles/how-to-take-meeting-notes)

4.3 **Constraints**

- Capture constraints such as:
  - Technical (stack, frameworks, infrastructure).  
  - Operational (SLA, timelines, budgets).  
  - Organizational (teams, policies).  

4.4 **Integration points with existing code**

- Identify references to:
  - Existing services, modules, APIs, or databases that must be reused or extended.  
  - Migration from legacy components.  
- Summarize integration expectations and impacts (for example, “Integrate with the existing billing service.”).  

4.5 **Discarded items**

- Capture ideas or proposals explicitly rejected or abandoned.  
- Note:
  - What was discarded.  
  - Optionally, a short reason if mentioned (“too complex”, “low value”).  

4.6 **Items deferred to future phases**

- Capture topics postponed or explicitly moved to future phases.  
- Include:
  - What is postponed.  
  - If available, to which phase or milestone.  

4.7 **Secondary technical details**

- Capture technical details that:
  - Are useful for context.  
  - Do not fundamentally change requirements, constraints, or decisions.  
- Examples:
  - Naming options, minor implementation preferences, to-do notes, and open questions that are not yet decisions. [plan](https://plan.io/blog/meeting-notes/)

The extraction must prioritize **signal over noise**: do not reproduce the full transcript or conversation; focus on the items relevant to these categories. [monday](https://monday.com/blog/productivity/take-better-meeting-notes/)

### Step 5: Build Structured Summary

Generate the final summary with the seven sections in this fixed order:

- `## Closed decisions`  
- `## Explicit functional requirements`  
- `## Constraints`  
- `## Integration points with existing code`  
- `## Discarded items`  
- `## Items deferred to future phases`  
- `## Secondary technical details`  

Under each heading:

- Use bullet lists (`-`) for each item.  
- Keep each bullet concise and unambiguous.  
- Make clear references to parts of the transcript only when necessary for disambiguation (for example, “as discussed in the section about integration with service X”).  

If a section is empty:

- Include a placeholder bullet, e.g.:  
  - `- No items identified for this category in the transcript.`  

Ensure that:

- The entire summary is in the language dictated by the global rules (typically Brazilian Portuguese in your setup, but this spec itself is language-agnostic).  
- The tone is professional, without slang, and with consistent terminology.  
- The summary does not introduce new decisions or requirements not supported by the transcript.

### Step 6: Highlight Ambiguities and Gaps (Optional)

When the transcript is unclear or contradictory:

- Prefer to reflect that ambiguity in the summary rather than resolving it.  
- Example phrasing:
  - `- There is disagreement in the transcript about this point; one part of the discussion suggests A, another suggests B. This requires an explicit decision in a future meeting.`  

Ambiguities and open questions can be placed in:

- The most relevant category (for example, constraints), or  
- “Secondary technical details” when they are not yet formal decisions or requirements.

***

## RULES

**Always:**

- Treat `transcription.md` (or the resolved transcript file) as the primary source of truth.  
- Produce a summary with all seven required sections, even if some are empty. [meetminutes](https://www.meetminutes.in/guide/meeting-templates)
- Keep the output concise, structured, and focused on decisions, requirements, constraints, integration points, discarded items, deferred items, and secondary technical details.

**Never:**

- Modify the transcript file or any other file.  
- Invent decisions, requirements, or constraints not supported by the transcript.  
- Omit any of the seven sections, even when they are empty.  
- Turn this summary into another document type (PRD, RFC, ADR, etc.); this skill only summarizes meeting transcripts.
