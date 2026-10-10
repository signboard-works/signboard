# 0019. Audit trail

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

> **Draft for review.** Written from Claude's recommendations while the decisions below were still open. Each one is listed under [Questions for review](#questions-for-review).

## Context

Signboard holds a shop's quotes, prices, jobs and invoices, and is shared by several people and, for the hosted service, operations staff ([ADR 0004](0004-multi-tenant.md), [ADR 0010](0010-api-first.md)). When a price changes, a job moves or someone gains access to an account, the business needs to know who did it, when, and what the object looked like afterwards. Earlier decisions already rely on an audit log: operations users adding themselves to an account ([ADR 0010](0010-api-first.md)) and deletions ([ADR 0018](0018-soft-delete.md)).

An audit trail is only worth anything if it cannot be bypassed. Application-level hooks miss whatever does not go through them: EF Core's bulk `ExecuteUpdate` and `ExecuteDelete` skip its change tracking, as does raw SQL and data migrations.

## Decision

One audit trail, provided centrally for every object and every table.

**What is recorded:**

- **Every change to every object:** create, update, soft delete and restore.
- **Security events:** sign-ins and failed sign-ins, membership and role changes, API clients and tokens being created, revoked or used, and operations users adding themselves to an account.
- **Reads, in one case only:** what an operations user views while a member of a business's account. Other reads are not audited.

**Each entry holds:**

- the object ID and type, and the tenant it belongs to;
- the actor: the user and the membership they acted in, an API client, or the system (background jobs, imports);
- the time, and a correlation ID shared by everything one request changed;
- the kind of event;
- a **snapshot of the whole object as JSON, as it was after the change**. Before-and-after values are not stored. A change is shown by comparing an entry's snapshot with the previous snapshot of the same object, which the UI does when it displays history.

**How it is captured:** PostgreSQL triggers on every table. One generic trigger function writes the row's new state as JSON. The application sets the actor and correlation ID per transaction, in the same way it sets the tenant for row-level security ([ADR 0004](0004-multi-tenant.md)), and the trigger reads them. Writes cannot reach a table without being audited, whichever code path makes them. Security events and audited reads are written by the application through one shared component.

**How it is stored:**

- One audit table, partitioned by month.
- **Append-only:** the application's database role can insert entries, but cannot update or delete them.
- Row-level security applies, so a business only ever sees its own trail. Platform events (the operations account, platform-level users) are in the platform's own trail.
- Retention: entries are kept for as long as the business exists, and deleted with it if it leaves the hosted service.

**How it is read:** through the public API, as an RQL-queryable resource ([ADR 0016](0016-rql-queries.md)), with a history view per object. Field visibility applies to snapshots: fields the actor may not see are removed from every snapshot **before** any comparison, so a customer user can never learn that a hidden field changed.

- Business administrators see their business's full trail.
- Customer users see a filtered history of their own objects (for example, a quote being sent, viewed and approved).
- Operations users see the platform's trail, and a business's trail only while they are members of its account.

**Privacy:** snapshots contain personal information. When personal information is de-identified ([ADR 0018](0018-soft-delete.md)), the de-identification process also clears it from that person's past snapshots. This is the only permitted change to existing entries, it is done by a separate, privileged process, and it is itself audited.

## Options considered

1. **Database triggers, full snapshot after each change**: cannot be bypassed; one generic mechanism for every table; history and diffs from consecutive snapshots. Logic lives in SQL, and snapshots use more storage than diffs. **Proposed.**
2. **EF Core interceptor in the application**: C#, familiar and easy to test; misses bulk updates, raw SQL and migrations, each a silent hole.
3. **Before-and-after values per changed field**: smaller entries; reconstructing an object's state at a point in time means replaying changes, and diffs are fixed at write time.
4. **Event sourcing**: the event log is the source of truth; a far larger architectural commitment than Signboard needs.
5. **Change data capture to an external store** (logical replication): nothing touches the write path; another system to run, and actor context is harder to carry.

## Consequences

- Every table gets the audit trigger, and an automated test fails if one is missing, alongside the row-level security check ([ADR 0012](0012-testing.md)).
- Snapshots are taken per table row. An object stored across several tables (a quote and its lines) produces one entry per row changed, linked by the correlation ID; the UI assembles them. Whether to snapshot whole aggregates instead is a question for review.
- Storage grows with every change. Monthly partitions keep queries and any future archiving manageable.
- The audit trail holds personal information, so it is in scope for privacy requests and access controls like any other data.
- An outbound feed to an external log or security monitoring service can be added later without changing how entries are captured.

## Questions for review

1. Scope: are security events in, and are reads audited only for operations users?
2. Is the audit trail one central mechanism (as drafted), or a pluggable interface with other stores behind it?
3. Is the one exception to append-only, de-identification of past snapshots, acceptable?
4. Visibility: business administrators, customer users and operations users as drafted?
5. Retention: for the life of the business, or a fixed period (for example seven years)?
6. Snapshots per table row (as drafted), or of the whole object across its tables (a quote with all its lines), which would need the application to build them?
