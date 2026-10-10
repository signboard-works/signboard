# Signboard: working notes for Claude

Signboard is open-source, self-hosted job management for sign and print shops: quotes, sales orders, jobs, a production kanban, invoices and accounting sync.

## Current phase

**Documentation first.** No application code until the PRDs and design docs for a phase are approved. Do not scaffold application code unless explicitly asked.

## Where things are

- `docs/product/`: vision and PRDs (user stories with Given/When/Then acceptance criteria)
- `docs/design/`: technical design documents
- `docs/adr/`: Architecture Decision Records (one decision each, numbered, never rewritten; supersede instead)
- `docs/templates/`: templates for new PRDs, design docs and ADRs
- `docs/glossary.md`: trade terms; add any new term used in a doc
- `docs/product/open-questions.md`: questions for the design partner shop

## Conventions

- British English spelling throughout.
- Use standard industry terms: PRD, design document, ADR, user story, acceptance criteria. Do not use SoftwareOne-internal acronyms (BDR, TDR).
- User story IDs use a module prefix: QUO, SO, JOB, INV, EML, MIG.
- Every doc has a status header: Draft, In review, Approved, Superseded.
- Diagrams in Mermaid, not images.
- Never put the design partner shop's name, staff names, pricing or customers in the repo. Refer to it as "the design partner shop".
- Shop-specific behaviour is configuration, never code (ADR 0006).
- Code examples always use curly braces, even for single-line blocks.

## Commands

```bash
pip install -r requirements-docs.txt
mkdocs serve          # preview at http://127.0.0.1:8000
mkdocs build --strict # check for broken links
```

## Git

Commit or push only when Stuart asks.
