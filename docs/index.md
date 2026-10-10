# Signboard

Open-source job management for sign and print shops.

Signboard is being **documented before it is built**. This site holds the product requirements, technical design and architecture decisions that the code will follow.

## How the docs are organised

| Section | What it holds | Audience |
|---|---|---|
| [Product](product/vision.md) | Vision, principles and Product Requirements Documents (PRDs) with user stories and acceptance criteria | Anyone who wants to know *what* Signboard does and *why* |
| [Design](design/architecture.md) | Technical design documents: architecture, data model, integrations, deployment | Contributors who want to know *how* it works |
| [Decisions](adr/index.md) | Architecture Decision Records (ADRs): one significant decision per record, with context and consequences | Anyone asking "why did we do it this way?" |
| [Glossary](glossary.md) | Sign and print trade terms, and Signboard's own vocabulary | Everyone, especially developers new to the trade |

## Document status

Every document carries a status in its header:

- **Draft**: being written; expect change.
- **In review**: complete enough for feedback.
- **Approved**: agreed; changes go through a pull request with a reason.
- **Superseded**: replaced by a newer document, which it links to.

## Contributing to the docs

Docs change through pull requests, exactly like code. Templates for new documents are in [`docs/templates/`](https://github.com/signboard-works/signboard/tree/main/docs/templates).
