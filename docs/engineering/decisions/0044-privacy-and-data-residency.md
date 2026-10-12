# 0044. Privacy and data residency

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-12 |
| **Deciders** | Stuart Meeks |

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
| A business's data is exportable at any time, and is deleted after it leaves | [ADR 0051](0051-data-export-and-leaving.md) |

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

- **The hosted service handles every business's data as if the APPs apply**, whatever the business's turnover. One standard is simpler to build and explain than following each business's status. Signboard's own operator may also be exempt by turnover, so this is a choice, not only a legal duty.
- **Self-hosted installations** are run by their operator, who is responsible for them. Signboard the project receives nothing from them ([ADR 0024](0024-product-analytics.md)).

### Data residency

- All business and personal data is stored and processed in **Australia**: in AWS Sydney, with backup copies in AWS Melbourne ([ADR 0030](0030-hosting-aws.md), [ADR 0043](0043-backups-and-recovery.md)).
- **Sub-processors** are listed publicly and kept current:

| Service | What it receives | Where |
|---|---|---|
| AWS (database, files, email, backups, logs) | All hosted data | Sydney; backup copies in Melbourne |
| AWS CloudFront | The static UI only; API responses are never cached ([ADR 0030](0030-hosting-aws.md)) | Global edge |
| PostHog | Usage events identified by user and account IDs, with no other business or personal data ([ADR 0024](0024-product-analytics.md)) | EU |
| GitHub, CodeRabbit | Source code and public issues, never business data ([ADR 0040](0040-backlog-and-issues.md)) | United States |

- **PostHog is the one exception to keeping data in Australia.** Its events carry no names, emails or business data, but they are identified by user and account IDs, which Signboard can link back to a person. They are therefore treated as personal information: the privacy policy discloses the transfer to the EU as an overseas disclosure. Notice alone does not meet APP 8.1, which requires reasonable steps to ensure the recipient handles the information as the APPs require, so before any event is sent PostHog's data processing agreement is signed, checked against APP 8.1 as part of the lawyer's review, and recorded. De-identifying a user also deletes their person record in PostHog together with all of their events. The erasure test covers it.
- An accounting system a business connects, such as Xero ([ADR 0005](0005-accounting-as-plugin.md)), receives data **at the business's direction**, under the business's own agreement with that provider. The connection screen says so.
- A new sub-processor that receives business or personal data needs an update to this ADR first, and notice to businesses. The next one expected is **Stripe**, when billing moves to it ([ADR 0047](0047-sign-up-plans-and-billing.md)): it would hold business administrators' billing details outside Australia, a second exception to residency, decided and disclosed before it is used.

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
| Backups | As set in [ADR 0043](0043-backups-and-recovery.md) | |
| Exports | 7 days | [ADR 0051](0051-data-export-and-leaving.md) |
| A business's data after it leaves | Deleted after 30 days read-only | [ADR 0051](0051-data-export-and-leaving.md) |

Keeping records such as invoices for as long as tax law requires is the business's obligation. Signboard keeps them while the business is a customer, and the export ([ADR 0051](0051-data-export-and-leaving.md)) lets the business keep them afterwards.

### Erasing personal information

- Erasure uses **de-identification**, as [ADR 0018](0018-soft-delete.md) decided: personal fields are cleared or replaced, such as with "Former contact", while the row, its ID and its references remain, so invoices and history still add up.
- **The ID that remains identifies no one.** Every field that could identify the person is cleared, IDs are random and never reused ([ADR 0017](0017-object-ids.md)), and the links to the person elsewhere are severed: their person record and events in PostHog are deleted, and logs and traces, which hold IDs only, expire within 90 days ([ADR 0025](0025-observability.md)). A test checks that no personal field survives de-identification.
- De-identification also clears the person from **audit snapshots** ([ADR 0019](0019-audit-trail.md)), from the **data snapshots kept with sent documents** ([ADR 0048](0048-documents-and-templates.md#kept-exactly-as-sent)) and from **search indexes** ([ADR 0045](0045-search.md)).
- **Files** that may contain the person, such as artwork or signed approvals, are listed for the business to review, because a file's content cannot be edited automatically. The erasure is not complete until each listed file has been deleted or replaced with a redacted copy, and until then the files are excluded from exports ([ADR 0051](0051-data-export-and-leaving.md)).
- **Backups** keep erased data until they expire ([ADR 0043](0043-backups-and-recovery.md)), up to 35 days. A restore can never bring erased information back, because every restore applies the [erasure log](0043-backups-and-recovery.md#the-erasure-log) first.
- Every erasure is audited, without recording the erased values.
- We do not use per-person encryption keys ("crypto-shredding"). They would make every query and search harder, for a guarantee that de-identification and expiring backups already give.

### Requests from individuals

- **A business's customers** ask the business, which is responsible for their information. Signboard gives business administrators the tools: find everything about a customer or contact, export it, correct it, and de-identify it (APPs 12 and 13).
- **Users** manage their own account data: see it, correct their name and email, download it, and close their account.
- **Closing a user account takes 30 days.** The user is emailed when they ask and 7 days before it takes effect, and signing in during those 30 days cancels it. After 30 days it is final: it ends the account's sessions and memberships, and deletes what the account itself holds: its email address, passkeys, passwords, second factors and API tokens ([ADR 0021](0021-authentication.md)). The person's **name stays on each business's records**, such as who approved a quote or moved a job, because those records are the business's, and who did what is part of them. Signboard keeps the name, and nothing else, on the closed user record so those references still read correctly.
- **People without access are marked, not hidden.** Wherever a business's records show a person who has no active membership of that business, because their account was closed or they were removed, the name is shown with **"(Deactivated)"**, such as "Jane Doe (Deactivated)". The label comes from the catalogue, like every other string ([ADR 0037](0037-localisation.md)). The API returns the name and the user's status, not the label.
- **Someone who returns.**
    - A person **removed from a business** whose account still exists keeps the same user ID. If the business invites them again, they get a new membership with the roles chosen by whoever invites them, the old one stays as history, and their past records link up again and lose the "(Deactivated)" mark.
    - A person who **closed their account** and signs up again, even with the same email address, is a **new user** with a new ID. Nothing links the two: re-linking would need the email address, or something derived from it, that closing the account deleted. Linking them by hand would let someone claim another person's history and would rewrite the append-only audit trail ([ADR 0019](0019-audit-trail.md)). The old records keep showing "Jane Doe (Deactivated)".
- **Removing the name** is the business's decision. If a former member asks, a business administrator de-identifies them with the tools above, and the records then show "Former user" with the same user ID. A request that reaches Signboard is passed to each business that holds the name.
- A request that reaches Signboard about a business's customer is passed to that business.

### Export and leaving

A business's export, and what happens to its data when it leaves, are decided in [ADR 0051](0051-data-export-and-leaving.md). A business that leaves is hard-deleted after 30 days read-only, and its closure is recorded in the [erasure log](0043-backups-and-recovery.md#the-erasure-log), so a restore never brings it back.

### Breaches

- Signboard's incident runbook ([ADR 0039](0039-security-baseline.md)) includes the Notifiable Data Breaches assessment, completed as fast as possible and within 30 days.
- **Affected businesses are told within 72 hours** of Signboard confirming that their data was involved, with what is known, so they can meet their own obligations to their customers. Signboard notifies the OAIC and affected users for the information it is responsible for, and coordinates with businesses on the rest.

### Policy and terms

- A **privacy policy** for the hosted service, and **terms of service** that include the data-handling commitments in this ADR, the sub-processor list and breach notice, are published before the first paying business.
- They are drafted from a reputable template and this ADR, then **a lawyer reviews them before the first paying business**, so the cost is a review, not a draft.

## Options considered

### Erasure

1. **De-identify fields, including audit snapshots and search indexes, and let backups expire**: keeps references and history intact, uses mechanisms already decided. **Chosen.**
2. **Hard-delete the person's rows**: simplest to explain; breaks invoices, history and object IDs ([ADR 0017](0017-object-ids.md)).
3. **Crypto-shredding** (a key per person, destroyed on erasure): erasure reaches backups at once; every query, index and search must decrypt, at large cost and complexity.

### Former users on a business's records

1. **Keep the name, marked "(Deactivated)"**: a business's history keeps saying who did what, and shows who no longer has access; the business can still remove the name on request. **Chosen.**
2. **Replace the name with "Former user" everywhere when the account closes**: removes the most; erases who did what from records that belong to the business, without the business deciding.

## Consequences

- The `system` module gains an erasure process, with its log for restores ([ADR 0043](0043-backups-and-recovery.md#the-erasure-log)). It needs tests, including cross-tenant cases ([ADR 0012](0012-testing.md)).
- Every design doc lists the personal information its feature collects, and how de-identification reaches it.
- A public sub-processor list, privacy policy and terms are tasks before the first paying business, alongside the incident runbook. The lawyer's review is the one cost this ADR adds.
- A user's status (active or closed) and a member's status are visible wherever a person is shown, so the UI can mark people "(Deactivated)".
