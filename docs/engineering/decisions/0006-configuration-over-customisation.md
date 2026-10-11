# 0006. Shop-specific behaviour is configuration, not code

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard is built with one design partner shop but must suit any sign or print shop. Shops differ in pricing rules, production stages, tax, payment terms, branding and document wording.

## Decision

We will treat everything that differs between shops as configuration: rates and pricing rules, work centres and kanban stages, tax settings, payment terms, numbering formats, branding, and email and document templates. No shop's specifics are hard-coded.

## Consequences

- A new shop can adopt Signboard without changing code.
- More upfront design for settings screens and sensible defaults.
- A shared starter configuration for a typical sign shop becomes a useful asset.
