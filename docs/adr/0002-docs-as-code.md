# 0002. Documentation as code, in the repository

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard will be fully specified before it is built: product requirements, technical design and decisions. The docs must stay accurate as the code evolves, be available to anyone who forks the project, and be usable as working context during development.

## Decision

We will keep all documentation as Markdown in the `docs/` folder of the main repository, published as a site with MkDocs Material. Diagrams are written in Mermaid. Documentation changes go through pull requests, alongside the code they describe.

## Options considered

1. **Markdown in the repo (MkDocs Material)**: versioned with the code, reviewable, forkable, diagrams as text. Weaker for non-technical commenting.
2. **Hosted wiki (Notion, Confluence)**: polished editing and commenting; separate from the code, not forkable, closed.
3. **Self-hosted wiki (Outline, BookStack)**: closest to a traditional wiki; another server to run.
4. **GitHub Wiki**: zero setup; no pull request review, poor versioning alongside code.

## Consequences

- Docs and code can be changed in the same pull request.
- Non-developer stakeholders review the published site and give feedback through issues or conversation rather than editing directly.
- Contributors need basic Markdown and Git.
