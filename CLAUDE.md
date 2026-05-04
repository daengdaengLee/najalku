# najalku

A study repository for Kubernetes and related components.

## Purpose

This repository is for studying Kubernetes and its surrounding components (e.g., descheduler).
Each component has its own forked repository, registered here as a git submodule for unified management.

## Seminar Materials

Seminar documents are stored in `./seminar/` and written in Markdown.
Naming convention: `NNNN-<topic-slug>.md` (e.g., `0001-k8s-scheduler-topology-spread.md`)
Seminar documents are written in Korean.

## Seminar Writing Guide

### Structure and Narrative
- Default to a technical-exposition arc tailored for seminar materials:
  problem & context → one-line thesis (BLUF) → mechanism (causal flow) → boundaries (what it does / does not do) → takeaway (and next bridge if part of a series)
- Do not adopt the decision-memo arc (past → change → current decision → future) — seminars explain mechanisms, not propose decisions
- Open with a clear problem statement and the BLUF; close with a "마무리" section
- Documents must be self-contained — readable without a presenter
- Favor restraint over exhaustive coverage: if a section grows long, split it into a follow-up unit rather than padding the current document

### Prose Style
- Write in complete sentences, organized into paragraphs that build a logical argument
- Use cause-and-effect connectors actively: 따라서, 그 결과, 그러나, 그래서, 즉
- Bullet points are reserved for genuinely list-like content (enumerations of equal-weight items, sequential steps); do not use bullets to fragment continuous reasoning
- Sentence endings: full polite/declarative endings (e.g., -다/-이다/-한다); do not use noun-terminal or 함/음 endings
- Do not use bold, italic, or blockquote for emphasis or structuring

### Visuals and Tables
- Use diagrams actively in the body
  - ASCII art with Unicode box-drawing glyphs and tree-branch glyphs
  - Wrap in untagged code fences; do not use Mermaid or images
- Prefer moving detailed tables to a "부록" (Appendix) section; keep only essential summary tables in the body
- Code fence language tags are reserved for actual code blocks (e.g., `yaml`, `go`, `bash`); leave diagrams untagged
- Inline code is allowed only when referencing actual code identifiers (function names, struct names, file names, etc.)

### Headings and Cross-References
- Use Markdown headings to separate sections and subsections
- Do not name section numbers in cross-references in any context unless explicitly instructed by the user

## Communication Rules

- **Documents, code, and comments**: write in **English** (seminar documents excluded — see Seminar Materials)
- **Conversation with the user**: use **Korean**
- **Explicit language instruction**: follow the instruction when given

## Notes for Claude

- Do not auto-commit. Always ask before committing.
- Keep submodule contents read-only unless the user explicitly requests changes to them.
