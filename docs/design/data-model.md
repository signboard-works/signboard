# Design: Data model

| | |
|---|---|
| **Status** | Draft |
| **Author** | Stuart Meeks |
| **Last updated** | 2026-10-10 |
| **Phase** | 1 |

## Context

Every entity, field and relationship, designed in full before build.

## Known constraints (to be expanded)

- Core chain: customer, quote (versions, lines, components), sales order, job, invoice, payment.
- Supporting: users and roles, catalogue items, materials, pricing rules, work centres, activity log, email events.
- Every object has one ID, a prefix and random digits in groups of four, which is also its primary key ([ADR 0017](../adr/0017-object-ids.md)). Versioned objects keep one ID; their versions are numbered beneath it.
- Rows are soft-deleted by default ([ADR 0018](../adr/0018-soft-delete.md)).

## Object ID prefixes

Every object type has one three-letter prefix and a fixed number of digits, a multiple of four ([ADR 0017](../adr/0017-object-ids.md)). Neither changes once the type is in use. Add a row here, with the digit length, when writing the type's requirements.

| Prefix | Object type | Digits |
|---|---|---|
| `USR` | User | To be decided |
| `ACC` | Account (business, customer or operations) | To be decided |
| `MEM` | Membership | To be decided |
| `QUO` | Quote | To be decided |
| `ORD` | Sales order | To be decided |
| `JOB` | Job | To be decided |
| `INV` | Invoice | To be decided |
| `EML` | Email | To be decided |
| `FIL` | File | To be decided |

This list grows as the data model is designed.

## Design

_To do._

## Failure modes

_To do._

## Alternatives considered

_To do._

## Open questions

_To do._
