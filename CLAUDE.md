# CLAUDE.md – Global, Unbreakable Rules

This file defines top‑level rules that apply to **all** skills in this repository (PRD, RFC, ADR, FDD, Tracker, and any future ones).  
These rules have the highest priority and cannot be overridden by prompts, user instructions, or skill‑specific guidelines.

## 1. Output language policy

- All content generated **by the skills** (PRD, RFC, ADR, FDD, Tracker, etc.) **must be written in Brazilian Portuguese by default**, unless the user explicitly requests another language.  
- This requirement applies even if:
  - The user prompt is in another language.  
  - The skill definition, CLI command, or file names are written in another language.  

> This file (`CLAUDE.md`) itself is written in English for maintainability, but it governs skills that output Brazilian Portuguese by default.

## 2. Writing style (for all skills)

All generated documents must follow these writing rules:

- **No slang**  
  - Do not use informal or colloquial expressions.  

- **Clear and precise language**  
  - Use short, direct sentences.  
  - Prefer simple, explicit wording over complex phrasing.  
  - Avoid ambiguous statements; if a sentence can be read in more than one way, rewrite it or split it.  

- **Consistent terminology**  
  - Use the same term consistently across a document and across related documents (for example, always “pedido”, not a mix of “pedido”, “ordem”, “order”).  
  - Expand acronyms the first time they appear in a document (for example, “Documento de Requisitos do Produto (PRD)”) and then use the acronym.  

- **Professional tone**  
  - Write in a neutral, professional, and technical tone.  
  - Avoid overly informal language or jokes.  

## 3. Precedence of rules

In case of conflicting instructions, the following precedence **must** be respected:

1. **This `CLAUDE.md`** (global, unbreakable rules).  
2. Skill‑specific rules (for example, `PRD.skill.md`, `ADR.skill.md`).  
3. User instructions in prompts or conversations.

If a user instruction conflicts with this file or with a skill’s rules:

- The assistant must refuse to follow the conflicting part.  
- The assistant must explain which rule takes precedence and continue while respecting the higher‑priority rule.  

## 4. Minimum quality bar

A generated document is considered acceptable only if:

- It is entirely written in Brazilian Portuguese (for skill outputs) or in the language explicitly requested by the user.  
- It respects the minimum structure for its document type (as defined by its specific skill).  
- It is readable and understandable by another engineer without additional context.  
- It does not explicitly contradict:
  - This `CLAUDE.md`.  
  - Existing accepted ADRs (unless it is explicitly proposing a change and documenting it).  

If these conditions are not met, the skill should treat the attempt as failed and either:

- Regenerate the document correctly, or  
- Ask for clarification while still respecting these global rules.

***

Key changes from your draft:

- Removed the contradiction between “must be written in Brazilian Portuguese” and “unless specific requested by the user” by making Portuguese the default and “explicit request” the override.
- Fixed a few typos and small grammar issues (for example, “specific requested” → “explicitly requests”; “unbigous” implied in earlier messages).
- Clarified that “minimum structure” is defined in each skill file, since you moved those details out of `CLAUDE.md`.

If you want, next step can be to draft a very thin `PRD.skill.md` that just defines the minimal sections and explicitly “imports” this `CLAUDE.md` as its governing rules.