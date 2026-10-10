# Signboard

Open-source job management for sign and print shops. Quotes, sales orders, production kanban and invoicing in one app: hosted, or run it yourself.

**Website:** [signboard.works](https://signboard.works)

> **Status:** Pre-alpha. The project is in its documentation phase: requirements and technical design are being written before any code. See [the docs](docs/index.md).

## What Signboard does

Signboard follows a job from the first enquiry to the final invoice:

1. **Quote**: price work using the methods sign shops actually use (area, machine time, per unit, labour, fixed).
2. **Sales order**: an accepted quote, locked, with deposits.
3. **Job**: production work routed through the shop's work centres.
4. **Kanban**: a live production board for the shop floor.
5. **Invoice**: billed from the sales order and synced to your accounting system.

Customer emails (quotes, proofs, invoices) are sent through properly authenticated infrastructure so they reach the inbox, not the junk folder.

## Why "Signboard"?

The name comes from *kanban* (看板), the Japanese word for a shop sign or signboard, and the production system that inspired Signboard's job board.

## Documentation

The docs live in [`docs/`](docs/index.md) and are built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

```bash
pip install -r requirements-docs.txt
mkdocs serve
```

## Licence

[Apache License 2.0](LICENSE). See [ADR 0003](docs/adr/0003-licence.md) for the reasoning.
