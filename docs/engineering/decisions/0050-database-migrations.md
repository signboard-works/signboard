# 0050. Database migrations

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-12 |
| **Deciders** | Stuart Meeks |

## Context

Schema changes are the commonest way a release damages data. Signboard has one shared database for every business ([ADR 0004](0004-multi-tenant.md)), so a bad migration hurts every business at once, and undoing it with a restore loses everything written since ([ADR 0043](0043-backups-and-recovery.md)).

The API and the UI deploy separately ([ADR 0015](0015-separate-deployments-one-origin.md)), and each API release must keep working with the previous UI build ([ADR 0029](0029-api-errors-idempotency-versioning.md)). During a deploy, old and new code run side by side. So a migration must never break the code running beside it. The application's database role cannot change the schema ([ADR 0004](0004-multi-tenant.md)), and several tasks start at once.

[ADR 0008](0008-database-postgresql.md) chose EF Core migrations, and now points here for how they run. This was first drafted as part of ADR 0043, and split out because it is a rule developers follow in every pull request, while backups are an operations concern.

## Decision

### Writing migrations

- **EF Core migrations**, reviewed as SQL in the pull request (`dotnet ef migrations script`). Every pull request with a migration shows its SQL.
- **Expand, then contract** ([ADR 0029](0029-api-errors-idempotency-versioning.md)). Each migration must work with the code already running:
    - a release only **adds**: new tables, new nullable or defaulted columns, new indexes;
    - removing or renaming is done in a later release, once no running code uses the old shape. A rename is an add, a backfill, a switch and, later, a drop;
    - nothing destructive happens in the same release as the code change that makes it possible.
- **Migrations must not block the database.** They set a lock timeout, create indexes concurrently, and avoid rewriting large tables. Backfilling data in a large table is a background job, in batches ([ADR 0034](0034-background-work.md)), not part of the migration.
- Database objects that tests depend on, such as row-level security policies and the revision, audit and outbox triggers ([ADR 0031](0031-concurrency.md), [ADR 0032](0032-events-webhooks-live-updates.md)), and seeded reference data, such as the units of measure ([ADR 0036](0036-money-tax-and-time.md#quantities)), are created only by migrations.

### Running migrations

- **Migrations run as their own step before the deploy**, as an EF Core migration bundle in a one-off ECS task, never when the application starts. The migration step has its own database role, used only by that step.
- **Deploy order:** migrations, then the API and worker, then the UI ([ADR 0015](0015-separate-deployments-one-origin.md)). If the migration fails, the deploy stops.
- **Self-hosted installations:** the Docker Compose file includes a `migrate` service that runs the same migration bundle before the application starts. The upgrade guide says to back up first ([ADR 0043](0043-backups-and-recovery.md#self-hosted-installations)).

### Testing migrations

- **On real PostgreSQL** with Testcontainers ([ADR 0012](0012-testing.md)): CI applies every migration from an empty database, and upgrades the previous release's schema with sample data in it.
- A check fails if the EF Core model has changes with no migration.

### Undoing a bad release

- **Fix forward, never roll back.** Down migrations are not used in production. If a release is bad, the previous application version still runs on the expanded schema, and a new migration fixes the problem.
- Point-in-time recovery ([ADR 0043](0043-backups-and-recovery.md)) is the last resort, because it loses everything since.

## Options considered

### Running migrations

1. **A migration bundle in a one-off task before the deploy**: one runner, its own database role, and the deploy stops if it fails. **Chosen.**
2. **On application start**: simplest; tasks race, and the application would need rights to change the schema.

### Undoing a bad release

1. **Fix forward, with expand-then-contract so the previous version still runs**: no data loss, and every change is reviewed. **Chosen.**
2. **Down migrations**: undo the schema; risky for data, rarely tested, and wrong once new data has been written.

### Where this is recorded

1. **Its own ADR**: one decision per record, easy to find and change apart from backups. **Chosen.**
2. **Together with backups in ADR 0043**: one place for everything about data loss; two decisions in one record, as review pointed out.

## Consequences

- The deployment design adds the migration step and the migration role, in the CDK code.
- Every pull request with a migration shows its SQL, follows expand-then-contract and passes the upgrade test. Migrations are a high-risk area that gets an adversarial review ([ADR 0041](0041-coding-with-ai.md#adversarial-review)).
- Removing a column or table takes two releases. That is deliberate.
