# 0043. Backups and disaster recovery

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-12 |
| **Deciders** | Stuart Meeks |

## Context

Every business's quotes, jobs, invoices and files live in one shared database and one file store ([ADR 0004](0004-multi-tenant.md), [ADR 0027](0027-file-storage.md)). Losing them, or restoring them wrongly, would hurt every business at once. Data can be lost in several ways, and each needs a different answer:

- **A mistake inside the app**, such as a bulk change that goes wrong for one business. Soft deletion ([ADR 0018](0018-soft-delete.md)) and the audit trail ([ADR 0019](0019-audit-trail.md)) cover most of these, but not all.
- **A bad release**, such as a migration that damages data. How migrations are written and run to avoid that is decided in [ADR 0050](0050-database-migrations.md).
- **An infrastructure failure**: a database instance, an availability zone, or rarely the whole Sydney region.
- **A compromised AWS account or credentials**, where an attacker, or a mistake, deletes the database and its backups together.

[ADR 0030](0030-hosting-aws.md) gives RDS automated backups for 14 days with point-in-time recovery, and moves the database to Multi-AZ before the first paying business. It left the recovery targets, copies outside the production account and restore testing to this ADR. All data must stay in Australia ([ADR 0030](0030-hosting-aws.md)), and AWS has a second Australian region, Melbourne (`ap-southeast-4`).

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
- **Backups cannot be deleted early.** The backup vault uses **AWS Backup Vault Lock**, so no one, including an attacker with administrator access to production, can delete a copy before it expires. It starts in **governance mode**, which can still be changed while the setup settles, and moves to **compliance mode**, which no one can change or remove, once the setup has run unchanged for a month. Compliance mode also fixes the 35-day retention, so it is checked before the switch.
- **No long-term archives.** There are no monthly or yearly backups. Businesses' records, including those kept for tax, are kept in the live database (and their accounting system) for as long as they need them; backups are for recovery only. This keeps personal data from living on in old backups ([ADR 0044](0044-privacy-and-data-export.md)).

### Files

- **S3 versioning** is turned on for the file bucket from launch, so a deleted or overwritten object can be recovered. Files never change ([ADR 0027](0027-file-storage.md)), so versioning costs almost nothing extra; older versions expire after 35 days.
- Before the first paying business, files are also backed up to the backup account in Melbourne through AWS Backup, under the same Vault Lock, kept for 35 days.

### The erasure log

A restore brings back the data as it was at the recovery point, including anything erased since ([ADR 0044](0044-privacy-and-data-export.md#erasing-personal-information)). The log of erasures and business closures that undoes that must therefore survive whatever the restore is recovering from, so it cannot live only in the database being restored.

- Each erasure or closure is written to the database in the same transaction as the change, and an outbox job ([ADR 0032](0032-events-webhooks-live-updates.md)) appends it to an **erasure log bucket** in S3, with Object Lock so entries cannot be changed or deleted.
- The bucket is **replicated to the backup account in Melbourne**, so the log survives the loss of the production account or the Sydney region. Replication starts with the other copies, before the first paying business.
- Entries hold IDs and what was erased or closed, never the erased values, so the log itself holds no personal information. They are kept for as long as any backup could predate them, at least 35 days, and in practice for the life of the service, since they are tiny.
- **Every restore applies the log before restored data reaches production or anyone using it:** a whole-database restore applies every entry later than its recovery point before traffic moves to it, and a single-business restore applies it to the exported data (below). The restore reads the log from the Melbourne copy when the Sydney copy is unavailable.

### Restoring one business

The database is shared, so a single business's data cannot be restored by rolling the whole database back. Instead:

1. Restore a snapshot from the right time to a temporary database, inside production.
2. Export that business's data from it, using the same **business export and import tool** that data export and moving to self-hosting use ([ADR 0044](0044-privacy-and-data-export.md)). One tool serves all three, so there is one thing to build and test.
3. Apply the [erasure log](#the-erasure-log) to the exported data, so nothing erased since the snapshot comes back.
4. Import it into production, reviewed and confirmed by an operations user, including recreating its files in object storage from the restored versions ([ADR 0027](0027-file-storage.md)), then delete the temporary database.

That tool is built before the first paying business, as [ADR 0004](0004-multi-tenant.md) requires. Restoring one business is an audited operations action ([ADR 0019](0019-audit-trail.md)).

### Restore testing

- **Monthly, automatically, from before the first paying business:** AWS Backup's restore testing restores the latest database snapshot to a temporary instance, a check confirms that it opens and that row counts and the latest migration are as expected, and the instance is deleted. A failure raises an alert ([ADR 0025](0025-observability.md)).
- **Every quarter, by hand, from the first paying business:** a drill that restores the database and files from the Melbourne copies, restores one business, checks that the erasure log, read from its Melbourne copy, was applied, and times both against the recovery targets. The drill is scripted, so it takes about an hour of the maintainer's time. The result is recorded in the journal ([ADR 0014](0014-build-journal.md)).
- **Production data never goes to staging or preview environments**, not even for a drill. Restores happen only in the production and backup accounts ([ADR 0039](0039-security-baseline.md)).

### Self-hosted installations

The self-hosting guide documents nightly backups: `pg_dump` in custom format and a copy of the file storage, kept on another machine or service, with a tested restore. An upgrade starts with a backup.

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

### How long copies are kept

1. **35 days, with no long-term archives**: covers a problem noticed weeks later, and erased personal data leaves backups within 35 days. **Chosen.**
2. **Monthly copies kept for a year**: recovers older mistakes; keeps erased personal data for a year and costs more.

### Restore testing

1. **Automated monthly test, plus a scripted quarterly drill once businesses pay**: proves the backups open every month for about A$3, and proves the whole procedure when it matters. **Chosen.**
2. **Manual drill only**: saves about A$3 a month; a broken backup could go unnoticed for a quarter.

## Consequences

- [ADR 0030](0030-hosting-aws.md) gains a fourth account, the backup account; backup copies in Melbourne, which stay in Australia; and about A$10 a month before paying businesses, taking that total to about A$475.
- The business export and import tool becomes a shared component, designed with [ADR 0044](0044-privacy-and-data-export.md).
- The deployment design adds the AWS Backup plans, the vault and its lock, the erasure log bucket and its replication, and the restore test, all in the CDK code.
- Backups hold personal data for up to 35 days after it is de-identified or deleted, which the privacy ADR and the privacy policy state.
- The switch of Vault Lock to compliance mode is a dated task on the board once the backup account is set up.
