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

- Use Markdown headings to separate sections and subsections
- Use diagrams actively
  - ASCII art with Unicode box-drawing glyphs and tree-branch glyphs
  - Wrap in untagged code fences; do not use Mermaid or images
- Use bullet points actively
- Use Markdown tables only when there is a clear need; prefer bullet points otherwise
- Code fence language tags are reserved for actual code blocks (e.g., `yaml`, `go`, `bash`); leave diagrams untagged
- Sentence endings: noun-terminal or simple endings (함/음 등)
- Do not use bold, italic, or blockquote for emphasis or structuring
- Inline code is allowed only when referencing actual code identifiers (function names, struct names, file names, etc.)
- Do not name section numbers in cross-references in any context unless explicitly instructed by the user

## Communication Rules

- **Documents, code, and comments**: write in **English** (seminar documents excluded — see Seminar Materials)
- **Conversation with the user**: use **Korean**
- **Explicit language instruction**: follow the instruction when given

## Notes for Claude

- Do not auto-commit. Always ask before committing.
- Keep submodule contents read-only unless the user explicitly requests changes to them.
