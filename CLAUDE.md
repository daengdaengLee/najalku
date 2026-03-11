# najalku

A study repository for Kubernetes and related components.

## Purpose

This repository is for studying Kubernetes and its surrounding components (e.g., descheduler).
Each component has its own forked repository, registered here as a git submodule for unified management.

## Structure

- `kubernetes/` — submodule: fork of the Kubernetes source (git@github.com:daengdaengLee/kubernetes.git)
- `descheduler/` — submodule: fork of the descheduler source (git@github.com:daengdaengLee/descheduler.git)
- `seminar/` — seminar materials in Markdown format

Each submodule has its own `CLAUDE.md` describing it as a study repository.

## Study Format

Weekly seminars of 15–20 minutes each. Each session:
1. Picks a specific component or feature as its topic
2. Covers its role and functionality
3. Dives into the actual source code to analyze behavior

## Seminar Materials

Seminar documents are stored in `./seminar/` and written in Markdown.
Naming convention: `NNNN-<topic-slug>.md` (e.g., `0001-k8s-scheduler-topology-spread.md`)

## Communication Rules

- **Documents, code, and comments**: write in **English**
- **Conversation with the user**: use **Korean**
- **Explicit language instruction**: follow the instruction when given

## Notes for Claude

- Do not auto-commit. Always ask before committing.
- When writing seminar materials, use English for the document body.
- Keep submodule contents read-only unless the user explicitly requests changes to them.
