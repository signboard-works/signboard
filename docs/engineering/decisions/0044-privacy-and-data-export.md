# 0044. Privacy, data residency and data export

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

> **Draft for review.** Written from Claude's recommendations. The choices still open are listed under [Questions for review](#questions-for-review).

## Context

Signboard holds personal information of three kinds: the users who sign in (names, email addresses), the businesses' customers and their contacts (names, emails, phone numbers, addresses), and whatever people write into quotes, jobs, notes and files. Every business shares one hosted service ([ADR 0004](0004-multi-tenant.md)).

Several decisions already touch privacy:

| Already decided | Where |
|---|---|
| Every service that stores data runs in Sydney | [ADR 0030](0030-hosting-aws.md) |
| Deletion is soft by default; personal information that must go is de-identified | [ADR 0018](0018-soft-delete.md) |
| The audit trail keeps snapshots, and de-identification clears them too | [ADR 0019](0019-audit-trail.md) (Proposed) |
| Analytics send IDs only, to PostHog's EU region; self-hosted sends nothing | [ADR 0024](0024-product-analytics.md) |
| Logs, traces and metrics hold IDs only, for 90 days | [ADR 0025](0025-observability.md) |
| No personal or business data in the repository or in AI tools | [ADR 0013](0013-work-in-the-open.md), [ADR 0041](0041-coding-with-ai.md) |
| An incident runbook before the first paying business; breach notification decided here | [ADR 0039](0039-security-baseline.md) |
| A business's data is exportable at any time, with no lock-in | [vision](../../vision.md) |

The law is the Privacy Act 1988 and its Australian Privacy Principles (APPs). Two points shape this ADR:

- **Many sign shops are exempt.** Businesses with annual turnover of A$3 million or less are generally outside the Act ([OAIC: small business](https://www.oaic.gov.au/privacy/privacy-guidance-for-organisations-and-government-agencies/organisations/small-business)). Signboard cannot rely on its customers' exemptions, and its customers' own customers deserve the same care either way.
- **The rules are tightening.** The Privacy and Other Legislation Amendment Act 2024 added a statutory tort for serious invasions of privacy from 10 June 2025, and transparency duties for automated decisions from 10 December 2026 ([the Act as made](https://www.legislation.gov.au/C2024A00128/asmade/text)).

The Notifiable Data Breaches scheme requires an entity that suspects an eligible data breach (one likely to cause serious harm) to assess it, taking all reasonable steps to finish within 30 days, and to notify affected individuals and the OAIC of an eligible breach ([OAIC: Notifiable Data Breaches scheme](https://www.oaic.gov.au/privacy/notifiable-data-breaches/about-the-notifiable-data-breaches-scheme); Privacy Act section 26WH).

This is a design decision, not legal advice.

## Decision

### Who is responsible for what

| Information | Responsible | Signboard's role |
|---|---|---|
| A business's customers, contacts and records | **The business** | Holds and processes it on the business's behalf, only to provide the service. Never uses it for anything else, never sells or shares it |
| Users' accounts (name, email, credentials, memberships) | **Signboard** (the hosted service's operator) | Responsible directly, because users are global and belong to no single business ([ADR 0010](0010-api-first.md)) |
| Product analytics and operational telemetry | **Signboard** | IDs only, never business or personal data ([ADR 0024](0024-product-analytics.md), [ADR 0025](0025-observability.md)) |

- **The hosted service handles every business's data as if the APPs apply**, whatever the business's turnover.
- **Self-hosted installations** are run by their operator, who is responsible for them. Signboard the project receives nothing from them ([ADR 0024](0024-product-analytics.md)).

### Data residency

- All business and personal data is stored and processed in **Australia**: in AWS Sydney, with backup copies in AWS Melbourne ([ADR 0030](0030-hosting-aws.md), [ADR 0043](0043-backups-recovery-and-migrations.md)).
- **Sub-processors** are listed publicly and kept current:

| Service | What it receives | Where |
|---|---|---|
| AWS (database, files, email, backups, logs) | All hosted data | Sydney; backup copies in Melbourne |
| AWS CloudFront | The static UI only; API responses are never cached ([ADR 0030](0030-hosting-aws.md)) | Global edge |
| PostHog | Usage events identified by user and account IDs, with no other business or personal data ([ADR 0024](0024-product-analytics.md)) | EU |
| GitHub, CodeRabbit | Source code and public issues, never business data ([ADR 0040](0040-backlog-and-issues.md)) | United States |

- **PostHog is the one exception to keeping data in Australia.** Its events carry no names, emails or business data, but they are identified by user and account IDs, which Signboard can link back to a person. They are therefore treated as personal information: the privacy policy discloses the transfer to the EU as an overseas disclosure under APP 8, and de-identifying a user also deletes their person record in PostHog together with all of their events. The erasure test covers it.
- An accounting system a business connects, such as Xero ([ADR 0005](0005-accounting-as-plugin.md)), receives data **at the business's direction**, under the business's own agreement with that provider. The connection screen says so.
- A new sub-processor that receives business or personal data needs an update to this ADR first, and notice to businesses.

### Collect less

- Each design doc lists the personal information its feature collects and why. A field without a reason is not added.
- No sensitive information (health, beliefs, government identifiers such as tax file numbers) is collected. Free-text fields say not to enter it.

### Retention

| Data | Kept | Decided in |
|---|---|---|
| Business records, including soft-deleted rows | For as long as the business uses the service | [ADR 0018](0018-soft-delete.md) |
| Audit trail | For as long as the business uses the service | [ADR 0019](0019-audit-trail.md) |
| Files, including sent documents | For as long as the business uses the service | [ADR 0027](0027-file-storage.md) |
| Logs, traces and metrics | 90 days | [ADR 0025](0025-observability.md) |
| Event outbox, completed jobs | 7 days | [ADR 0032](0032-events-webhooks-live-updates.md), [ADR 0034](0034-background-work.md) |
| Idempotency keys | 24 hours | [ADR 0029](0029-api-errors-idempotency-versioning.md) |
| Backups | As set in [ADR 0043](0043-backups-recovery-and-migrations.md) | |
| A business's data after it leaves | Deleted after the grace period below | This ADR |

Keeping records such as invoices for as long as tax law requires is the business's obligation. Signboard keeps them while the business is a customer, and the export lets the business keep them afterwards.

### Erasing personal information

- Erasure uses **de-identification**, as [ADR 0018](0018-soft-delete.md) decided: personal fields are cleared or replaced, such as with "Former contact", while the row, its ID and its references remain, so invoices and history still add up.
- **The ID that remains identifies no one.** Every field that could identify the person is cleared, IDs are random and never reused ([ADR 0017](0017-object-ids.md)), and the links to the person elsewhere are severed: their person record and events in PostHog are deleted, and logs and traces, which hold IDs only, expire within 90 days ([ADR 0025](0025-observability.md)). A test checks that no personal field survives de-identification.
- De-identification also clears the person from **audit snapshots** ([ADR 0019](0019-audit-trail.md)) and from **search indexes** ([ADR 0045](0045-search.md)).
- **Files** that may contain the person, such as artwork or signed approvals, are listed for the business to review, because a file's content cannot be edited automatically. The erasure is not complete until each listed file has been deleted or replaced with a redacted copy, and until then the files are excluded from exports.
- **Backups** keep erased data until they expire ([ADR 0043](0043-backups-recovery-and-migrations.md)). Every erasure is recorded in a durable erasure log, and the restore tooling applies that log to restored data **before** it reaches production, so a restore can never bring erased information back. Restore tests check it.
- Every erasure is audited, without recording the erased values.
- We do not use per-person encryption keys ("crypto-shredding"). They would make every query and search harder, for a guarantee that de-identification and expiring backups already give.

### Requests from individuals

- **A business's customers** ask the business, which is responsible for their information. Signboard gives business administrators the tools: find everything about a customer or contact, export it, correct it, and de-identify it (APPs 12 and 13).
- **Users** manage their own account data: see it, correct their name and email, download it, and close their account.
- **Closing a user account** ends its sessions and memberships, and de-identifies the user's name and email everywhere, including in each business's records and audit trail, which then show "Former user" with the same user ID. A business's record of what was done stays intact; who it was does not.
- A request that reaches Signboard about a business's customer is passed to that business.

### Data export

- **A business administrator can export everything at any time**, not only when leaving. It is an export job ([ADR 0034](0034-background-work.md)) that produces one download:
    - **JSON**, one file per collection, each object exactly as the API returns it to an administrator ([ADR 0020](0020-url-structure.md)), with the API version recorded;
    - **CSV** for the main collections (customers, quotes, orders, jobs, invoices), for spreadsheets;
    - **every file** in its original form, including the exact documents sent to customers ([ADR 0027](0027-file-storage.md));
    - the audit trail, as JSON.
- The export is free in every edition, never behind an entitlement ([ADR 0046](0046-entitlements.md)), and documented in the API reference so it can be read without Signboard.
- Exports are audited, available to administrators only, and deleted 7 days after they are created.

### A business leaving

1. An administrator closes the business. The business becomes **read-only for 30 days**, with export available, and administrators are emailed. A business that only stops paying becomes read-only but is not deleted for non-payment alone ([ADR 0047](0047-sign-up-plans-and-billing.md)).
2. After 30 days, the business's data is **hard-deleted**: database rows, files, search indexes and email records ([ADR 0018](0018-soft-delete.md), exception 1). Users remain, because they are global, but lose their memberships. The closure is recorded in the erasure log, so a restore from an earlier snapshot never brings a closed business back; restore tests cover it.
3. Backups age out on their own schedule. The business is told the date by which its data will be gone from backups too.

### Breaches

- Signboard's incident runbook ([ADR 0039](0039-security-baseline.md)) includes the Notifiable Data Breaches assessment, completed as fast as possible and within 30 days.
- **Affected businesses are told within 72 hours** of Signboard confirming that their data was involved, with what is known, so they can meet their own obligations to their customers. Signboard notifies the OAIC and affected users for the information it is responsible for, and coordinates with businesses on the rest.

### Policy and terms

- A **privacy policy** for the hosted service, and **terms of service** that include the data-handling commitments in this ADR, the sub-processor list and breach notice, are published before the first paying business.
- **A lawyer reviews them before the first paying business.** Drafting starts from this ADR.

## Options considered

### Erasure

1. **De-identify fields, including audit snapshots and search indexes, and let backups expire**: keeps references and history intact, uses mechanisms already decided. **Chosen.**
2. **Hard-delete the person's rows**: simplest to explain; breaks invoices, history and object IDs ([ADR 0017](0017-object-ids.md)).
3. **Crypto-shredding** (a key per person, destroyed on erasure): erasure reaches backups at once; every query, index and search must decrypt, at large cost and complexity.

### Export format

1. **JSON as the API returns it, CSV for the main collections, and the original files**: complete, documented and readable without Signboard; matches the API, so no second format to maintain. **Chosen.**
2. **A database dump**: complete; exposes internal tables and is useless without Signboard's schema.
3. **CSV only**: easy to open; loses structure, such as quote lines and versions.

## Consequences

- The `system` module gains export jobs, an erasure process (with its log for restores) and business closure. Each needs tests, including cross-tenant cases ([ADR 0012](0012-testing.md)).
- Every design doc lists the personal information its feature collects, and how de-identification reaches it.
- A public sub-processor list, privacy policy and terms are tasks before the first paying business, alongside the incident runbook. The lawyer's review is the one cost this ADR adds.
- The API reference documents the export's structure.

## Questions for review

1. **Treat every business's data as covered by the APPs**, even though many sign shops are exempt by turnover. Or follow each business's own status?
2. **De-identification, not crypto-shredding**, with backups allowed to expire naturally and a log of erasures re-applied after any restore. Agreed?
3. **Closing a user account de-identifies the user in every business's records**, which then show "Former user". Or should businesses keep the names of former staff on their own records?
4. **Export at any time**, as JSON plus CSV plus files, free in every edition. Agreed?
5. **30 days read-only after a business leaves**, then hard deletion. Longer, such as 90 days, or shorter?
6. **Tell affected businesses within 72 hours** of confirming their data was involved in a breach. Agreed, or a different window?
7. **A lawyer reviews the privacy policy and terms** before the first paying business, which is a cost. Or start from a reputable template and review later?
8. **One ADR or several.** This ADR covers roles, residency, retention, erasure, export, leaving and breaches together, as scoped in issue #37, because they share one subject: what happens to people's data. Keep it whole, or split it?
