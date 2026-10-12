# Design: Data model

| | |
|---|---|
| **Status** | Draft |
| **Author** | Stuart Meeks |
| **Last updated** | 2026-10-12 |
| **Phase** | 1 |

## Context

Every entity, field and relationship, designed in full before build.

## Known constraints (to be expanded)

- Core chain: customer, quote (versions, lines, components), sales order, job, invoice, payment.
- Supporting: users and roles, catalogue items, materials, pricing rules, work centres, activity log, email events.
- Search documents, one per object and field group, in the `system` module ([ADR 0045](../decisions/0045-search.md)).
- Platform reference data, shared by every business and managed by operations users: units of measure ([ADR 0036](../decisions/0036-money-tax-and-time.md#quantities)).
- Every object has one ID, a prefix and random digits in groups of four, which is also its primary key ([ADR 0017](../decisions/0017-object-ids.md)). Versioned objects keep one ID; their versions are numbered beneath it.
- Rows are soft-deleted by default ([ADR 0018](../decisions/0018-soft-delete.md)).

## Object ID prefixes

Every object type has one three-letter prefix and a fixed number of digits, a multiple of four ([ADR 0017](../decisions/0017-object-ids.md)). Neither changes once the type is in use. Add a row here, with the digit length, when writing the type's requirements.

| Prefix | Object type | Digits |
|---|---|---|
| `USR` | User | To be decided |
| `ACC` | Account (business, customer or operations) | To be decided |
| `MEM` | Membership | To be decided |
| `SUB` | Subscription to the hosted service ([ADR 0047](../decisions/0047-sign-up-plans-and-billing.md#subscription-status)) | To be decided |
| `QUO` | Quote | To be decided |
| `ORD` | Sales order | To be decided |
| `JOB` | Job | To be decided |
| `INV` | Invoice | To be decided |
| `EML` | Email | To be decided |
| `FIL` | File | To be decided |
| `TPL` | Document or email template ([ADR 0048](../decisions/0048-documents-and-templates.md)) | To be decided |
| `EVT` | Event ([ADR 0032](../decisions/0032-events-webhooks-live-updates.md)) | To be decided |
| `WHK` | Webhook ([ADR 0032](../decisions/0032-events-webhooks-live-updates.md)) | To be decided |
| `UOM` | Unit of measure ([ADR 0036](../decisions/0036-money-tax-and-time.md#quantities)) | To be decided |

This list grows as the data model is designed.

## Design

_To do._

## Failure modes

_To do._

## Alternatives considered

_To do._

## Open questions

_To do._
