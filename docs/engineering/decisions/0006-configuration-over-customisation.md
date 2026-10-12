# 0006. Business-specific behaviour is configuration, not code

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard is built with one design partner shop but must suit any sign or print shop, and stay open to other made-to-order businesses ([ADR 0049](0049-scope-made-to-order-businesses.md)). Businesses differ in pricing rules, production stages, tax, payment terms, branding and document wording.

## Decision

We will treat everything that differs between businesses as configuration: rates and pricing rules, work centres and kanban stages, tax settings, payment terms, numbering formats, branding, and email and document templates. No business's specifics are hard-coded, and nor are any one trade's.

## Consequences

- A new business can adopt Signboard without changing code.
- More upfront design for settings screens and sensible defaults.
- A starter pack of configuration for each trade, beginning with sign and print, becomes a useful asset ([ADR 0049](0049-scope-made-to-order-businesses.md)).
