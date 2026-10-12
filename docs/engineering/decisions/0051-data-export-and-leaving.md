# 0051. Data export and leaving

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-12 |
| **Deciders** | Stuart Meeks |

## Context

The vision promises that a business owns its data: everything can be exported in open formats, with no lock-in, including to Signboard itself. Three things need a business's data out of Signboard and, in two cases, back in:

- a business **exporting** its data, at any time, for its own records or to leave;
- a business **moving to self-hosting**, or back;
- operations **restoring one business** from a backup ([ADR 0043](0043-backups-and-recovery.md#restoring-one-business)).

A business that leaves also needs a clear end: how long its data stays, how it is warned, and when it is gone. Privacy rules for that data are in [ADR 0044](0044-privacy-and-data-residency.md). This was first drafted as part of ADR 0044, and split out because it is a product feature that other decisions rely on, rather than a privacy rule.

## Decision

### Data export

- **A business administrator can export everything at any time**, not only when leaving. It is an export job ([ADR 0034](0034-background-work.md)) that produces one download:
    - **JSON**, one file per collection, each object exactly as the API returns it to an administrator ([ADR 0020](0020-url-structure.md)), with the API version recorded;
    - **CSV** for the main collections (customers, quotes, orders, jobs, invoices), for spreadsheets;
    - **every file** in its original form, including the exact documents sent to customers ([ADR 0027](0027-file-storage.md)), except files awaiting redaction after an erasure ([ADR 0044](0044-privacy-and-data-residency.md#erasing-personal-information));
    - the audit trail, as JSON.
- The export is free in every edition, never behind an entitlement ([ADR 0046](0046-entitlements.md)), and available while a business is read-only ([ADR 0047](0047-sign-up-plans-and-billing.md)).
- Its structure is documented in the API reference, so it can be read without Signboard.
- Exports are audited ([ADR 0019](0019-audit-trail.md)), available to administrators only, and deleted 7 days after they are created.

### One export and import tool

- The same **business export and import tool** serves exporting, moving between the hosted service and a self-hosted installation, and restoring one business from a backup. There is one format and one thing to build and test.
- Import keeps every object ID ([ADR 0017](0017-object-ids.md)), refuses data for a business that already exists unless an operations user is restoring it, and applies the [erasure log](0043-backups-and-recovery.md#the-erasure-log) before anything is written.
- The tool is built before the first paying business, as [ADR 0004](0004-multi-tenant.md) requires.

### A business leaving

1. An administrator closes the business. The business becomes **read-only for 30 days**, with export available, and administrators are emailed straight away, **7 days** before deletion and **1 day** before it.
2. During those 30 days an administrator can **cancel the closure**, and the business carries on as before.
3. After 30 days, the business's data is **hard-deleted**: database rows, files, search indexes and email records ([ADR 0018](0018-soft-delete.md), exception 1). Users remain, because they are global, but lose their memberships. The closure is recorded in the [erasure log](0043-backups-and-recovery.md#the-erasure-log), so a restore from an earlier snapshot never brings a closed business back; restore tests cover it.
4. Backups age out on their own schedule. The business is told the date by which its data will be gone from backups too.

A business that only stops paying becomes read-only but is never deleted for non-payment alone ([ADR 0047](0047-sign-up-plans-and-billing.md)).

## Options considered

### Export format

1. **JSON as the API returns it, CSV for the main collections, and the original files**: complete, documented and readable without Signboard; matches the API, so no second format to maintain. **Chosen.**
2. **A database dump**: complete; exposes internal tables and is useless without Signboard's schema.
3. **CSV only**: easy to open; loses structure, such as quote lines and versions.

### Time before deletion

1. **30 days read-only, with reminders and the option to cancel**: enough time to export and to change one's mind, without holding a departed business's data for long. **Chosen.**
2. **90 days**: more forgiving; holds data three times as long after the business has asked to leave.
3. **Immediate deletion**: the least data held; a mistaken closure cannot be undone.

## Consequences

- The `system` module gains export jobs, the import tool and business closure. Each needs tests, including cross-tenant cases ([ADR 0012](0012-testing.md)).
- The API reference documents the export's structure.
- The emails for closure and its reminders are platform emails ([ADR 0023](0023-email-sending.md)).
