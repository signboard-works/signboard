# 0043. Backups, disaster recovery and migrations

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

> **Draft for review.** Written from Claude's recommendations. The choices still open are listed under [Questions for review](#questions-for-review).

## Context

Every business's quotes, jobs, invoices and files live in one shared database and one file store ([ADR 0004](0004-multi-tenant.md), [ADR 0027](0027-file-storage.md)). Losing them, or restoring them wrongly, would hurt every business at once. Data can be lost in several ways, and each needs a different answer:

- **A mistake inside the app**, such as a bulk change that goes wrong for one business. Soft deletion ([ADR 0018](0018-soft-delete.md)) and the audit trail ([ADR 0019](0019-audit-trail.md)) cover most of these, but not all.
- **A bad release**, such as a migration that damages data.
- **An infrastructure failure**: a database instance, an availability zone, or rarely the whole Sydney region.
- **A compromised AWS account or credentials**, where an attacker, or a mistake, deletes the database and its backups together.

[ADR 0030](0030-hosting-aws.md) already gives RDS automated backups for 14 days with point-in-time recovery, and moves the database to Multi-AZ before the first paying business. It left the recovery targets, copies outside the production account and restore testing to this ADR. All data must stay in Australia ([ADR 0030](0030-hosting-aws.md)), and AWS has a second Australian region, Melbourne (`ap-southeast-4`).

Schema changes are part of the same problem. The API and the UI deploy separately, and each API release must keep working with the previous UI build ([ADR 0029](0029-api-errors-idempotency-versioning.md)), so a migration must never break the code running beside it.

Initial development is kept as cheap as possible, so everything here is priced from AWS's Sydney price list on 2026-10-11, at A$1.43 to the US dollar including GST, as in [ADR 0030](0030-hosting-aws.md).

## Decision

### Recovery targets

| Failure | Recovery point (data lost at most) | Recovery time (back in service within) |
|---|---|---|
| A database instance or availability zone fails | None after Multi-AZ; up to 5 minutes before it | Minutes after Multi-AZ; about 1 hour before it |
| A bad release or mistake damages data | 5 minutes (point-in-time recovery) | 4 hours |
| The Sydney region is lost, or the production account is compromised | 24 hours | 2 days |

These targets are stated publicly, so businesses know what to expect, and each restore drill checks them.

### The database

- **Automated backups and point-in-time recovery** stay as in [ADR 0030](0030-hosting-aws.md): kept for 14 days, with transaction logs every 5 minutes. RDS backup storage up to the size of the database is free.
- **A copy outside production.** Before the first paying business, alongside Multi-AZ, a daily copy of the database snapshot goes to a **separate backup account, in Melbourne**, through **AWS Backup**. Copies are kept for 35 days. The backup account is the fourth account in the organisation ([ADR 0030](0030-hosting-aws.md)); nothing in production can delete what is in it.
- **Backups cannot be deleted early.** The backup vault uses **AWS Backup Vault Lock**, so no one, including an attacker with administrator access to production, can delete a copy before it expires.
- **No long-term archives.** There are no monthly or yearly backups. Businesses' records are kept in the live database for as long as they need them; backups are for recovery only. This keeps personal data from living on in old backups ([ADR 0044](0044-privacy-and-data-export.md)).

### Files

- **S3 versioning** is turned on for the file bucket, so a deleted or overwritten object can be recovered. Files never change ([ADR 0027](0027-file-storage.md)), so versioning costs almost nothing extra; older versions expire after 35 days.
- Before the first paying business, files are also backed up to the backup account in Melbourne through AWS Backup, under the same Vault Lock, kept for 35 days.

### Restoring one business

The database is shared, so a single business's data cannot be restored by rolling the whole database back. Instead:

1. Restore a snapshot from the right time to a temporary database, inside production.
2. Export that business's data from it, using the same **business export and import tooling** that data export and moving to self-hosting use ([ADR 0044](0044-privacy-and-data-export.md)).
3. Apply the erasure log ([ADR 0044](0044-privacy-and-data-export.md#erasing-personal-information)) to the exported data, so nothing erased since the snapshot comes back.
4. Import it into production, reviewed and confirmed by an operations user, including recreating its files in object storage from the restored versions ([ADR 0027](0027-file-storage.md)), then delete the temporary database.

That tooling is built before the first paying business, as [ADR 0004](0004-multi-tenant.md) requires. Restoring one business is an audited operations action ([ADR 0019](0019-audit-trail.md)).

### Restore testing

- **Monthly, automatically:** AWS Backup's restore testing restores the latest database snapshot to a temporary instance, a check confirms that it opens and that row counts and the latest migration are as expected, and the instance is deleted. A failure raises an alert ([ADR 0025](0025-observability.md)).
- **Every quarter, by hand:** a full drill that restores the database and files from the Melbourne copies, restores one business, checks that the erasure log was applied, and times both against the recovery targets. The result is recorded in the journal ([ADR 0014](0014-build-journal.md)).
- **Production data never goes to staging or preview environments**, not even for a drill. Restores happen only in the production and backup accounts ([ADR 0039](0039-security-baseline.md)).

### Migrations

- **EF Core migrations**, reviewed as SQL in the pull request (`dotnet ef migrations script`).
- **Migrations run as their own step before the deploy**, as an EF Core migration bundle in a one-off ECS task, never when the application starts. Several tasks starting at once would race, and the application's database role cannot change the schema ([ADR 0004](0004-multi-tenant.md)). The migration has its own role, used only by that step.
- **Deploy order:** migrations, then the API and worker, then the UI ([ADR 0015](0015-separate-deployments-one-origin.md)).
- **Expand, then contract** ([ADR 0029](0029-api-errors-idempotency-versioning.md)). Each migration must work with the code already running:
    - a release only **adds**: new tables, new nullable or defaulted columns, new indexes;
    - removing or renaming is done in a later release, once no running code uses the old shape. A rename is an add, a backfill, a switch and, later, a drop;
    - nothing destructive happens in the same release as the code change that makes it possible.
- **Migrations must not block the database.** They set a lock timeout, create indexes concurrently, and avoid rewriting large tables. Backfilling data in a large table is a background job, in batches ([ADR 0034](0034-background-work.md)), not part of the migration.
- **Tested on real PostgreSQL** with Testcontainers ([ADR 0012](0012-testing.md)): CI applies every migration from an empty database, and upgrades the previous release's schema with sample data in it. A check fails if the EF Core model has changes with no migration.
- **Forward-fix, never roll back.** Down migrations are not used in production. If a release is bad, the previous application version still runs on the expanded schema, and a new migration fixes the problem. Point-in-time recovery is the last resort, because it loses everything since.
- Database objects that tests depend on, such as row-level security policies and the revision, audit and outbox triggers ([ADR 0031](0031-concurrency.md), [ADR 0032](0032-events-webhooks-live-updates.md)), are created only by migrations.

### Self-hosted installations

- The Docker Compose file includes a `migrate` service that runs the same migration bundle before the application starts.
- The self-hosting guide documents nightly backups: `pg_dump` in custom format and a copy of the file storage, kept on another machine or service, with a tested restore. An upgrade starts with a backup.

### Cost

| | A$ a month |
|---|---:|
| At launch (S3 versioning; RDS backups within the free allowance) | about 0 |
| Before paying businesses: copies to Melbourne (transfer at US$0.08 per GB, snapshot storage at US$0.095 per GB-month, S3 backup storage at US$0.06 per GB-month), a KMS key in the backup account, and monthly restore tests at US$1.80 each | about 10 |

The estimate assumes a database of about 5 GB and 20 GB of files, and grows with the data.

## Options considered

### Copies outside production

1. **AWS Backup copies to a separate account in Melbourne, with Vault Lock, from before the first paying business**: survives an account compromise and a regional outage, keeps all data in Australia, and costs about A$10 a month. **Chosen.**
2. **From launch**: protects test data that does not yet need it, for the same cost.
3. **Same-account, same-region backups only**: free; a compromised account or a mistake can delete the backups with the database.
4. **Copies to another country's region**: more distant from Sydney; breaks the rule that all data stays in Australia.

### Running migrations

1. **A migration bundle in a one-off task before the deploy**: one runner, its own database role, and the deploy stops if it fails. **Chosen.**
2. **On application start**: simplest; tasks race, and the application would need rights to change the schema.

### Undoing a bad release

1. **Forward-fix, with expand-then-contract so the previous version still runs**: no data loss, and every change is reviewed. **Chosen.**
2. **Down migrations**: undo the schema; risky for data, rarely tested, and wrong once new data has been written.

## Consequences

- **This changes [ADR 0030](0030-hosting-aws.md)**, which keeps every data-storing service, backups included, in Sydney. Backup copies in Melbourne stay in Australia, so the residency commitment holds, but ADR 0030 is updated to match when this ADR is accepted.
- [ADR 0030](0030-hosting-aws.md) gains a fourth account, the backup account, and about A$10 a month before paying businesses, taking that total to about A$475.
- The business export and import tooling becomes a shared component, designed with [ADR 0044](0044-privacy-and-data-export.md).
- The deployment design adds the migration step, the migration role, AWS Backup plans and the restore test, all in the CDK code.
- Every pull request with a migration shows its SQL, follows expand-then-contract, and passes the upgrade test.
- Backups hold personal data for up to 35 days after it is de-identified or deleted, which the privacy ADR and the privacy policy state.

## Questions for review

1. **The recovery targets**: 5 minutes and 4 hours for ordinary failures; 24 hours and 2 days for losing Sydney or the production account. Or stricter, at more cost?
2. **Copies to a backup account in Melbourne from before the first paying business**, about A$10 a month. Or from launch, or not at all until there is demand?
3. **Vault Lock**, so backups cannot be deleted early even by an administrator. Lean: yes, starting in governance mode, which can be changed, and moving to compliance mode, which cannot, once the setup has been stable for a month.
4. **35 days of copies and no long-term archives**, so deleted personal data leaves backups within 35 days. Or keep monthly copies for a year?
5. **Monthly automated restore tests and a quarterly manual drill.** Or only the manual drill, saving about A$3 a month?
6. **Migrations as a separate step before each deploy**, never on start-up. Agreed?
7. **Forward-fix only**, with no down migrations in production. Agreed?
8. **One business export and import tool** serving data export, moving to self-hosting and single-business restore. Agreed?
9. **Migrations in this ADR or their own.** This ADR covers backups, recovery and schema migrations together, as scoped in issue #36, because migrations are the commonest way data is lost. Keep them together, or split migrations into their own ADR?
