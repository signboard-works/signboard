# 0031. Concurrency

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

> **Draft for review.** Written from Claude's recommendations. The choices still open are listed under [Questions for review](#questions-for-review).

## Context

Several people work on the same objects at once. Two people may edit the same quote, a designer and a salesperson may change the same job, someone may drag a job on the kanban board while someone else reschedules it, and an integration may update a customer while a user has it open. The background worker changes objects too, such as when an email is delivered or an invoice syncs.

Without a rule, the last write wins silently: one person's changes overwrite another's, and nobody knows. That is worst where money is involved. Someone could send or approve a quote without having seen a price change made a minute earlier.

`PATCH` with JSON Merge Patch ([ADR 0020](0020-url-structure.md)) only sends the fields being changed, which narrows the problem but does not remove it: two people can change the same field, or one change can make another wrong (a discount approved against a total that has since changed). [ADR 0029](0029-api-errors-idempotency-versioning.md) reserved `412` and `428` for this decision.

## Decision

We will use **optimistic concurrency** throughout: nothing is locked while someone is editing. Every change says which version of the object it was based on, and is refused if the object has changed since.

### The concurrency token

- Every object (every aggregate root, such as a quote, job or customer) has a **concurrency version**: a number stored in a `concurrency_version` column that goes up by one on every change.
- A change to anything the object owns, such as a quote line, also increments the object's version. The version describes the whole object as the API returns it ([ADR 0020](0020-url-structure.md)), not one table row. An EF Core save interceptor does this, so it cannot be forgotten.
- We do not use PostgreSQL's `xmin` system column. It changes only for the row it belongs to, so it cannot cover child rows, and it does not survive a dump and restore.
- The name avoids "version", because quotes already have business versions numbered beneath their ID ([ADR 0017](0017-object-ids.md)).

### ETags and If-Match

- Every response that returns an object carries its version as a strong **`ETag`** header, such as `ETag: "17"`. Clients treat it as an opaque string.
- The object's JSON also carries it as a read-only **`etag`** field, so items in a collection, which have no headers of their own, can be edited from a list.
- **`PATCH`, `DELETE` and actions** (`POST` to a verb sub-path, such as `/send` or `/approve`) must send **`If-Match`** with the ETag they were based on.
    - Missing `If-Match`: `428 Precondition Required`, code `precondition_required`.
    - ETag does not match: `412 Precondition Failed`, code `precondition_failed`. The problem details include `currentEtag`, and the client fetches the object again.
    - A match: the change is applied and the response carries the new ETag.
- **Creates** (`POST` to a collection) need no `If-Match`. Duplicates are prevented by idempotency keys ([ADR 0029](0029-api-errors-idempotency-versioning.md)).
- **`If-Match: *`** ("whatever the current version is") is accepted, for integrations that deliberately want the last write to win. Signboard's own UI never sends it, and the audit trail records which credential made the change ([ADR 0019](0019-audit-trail.md)).
- **`If-None-Match`** on a `GET` of one object returns `304 Not Modified` when the ETag still matches, so clients can check cheaply whether an object has changed.

### In the database

- An update runs as `UPDATE … SET …, concurrency_version = concurrency_version + 1 WHERE id = @id AND concurrency_version = @expected`. If no row is updated, the object changed in the meantime and the API returns `412`. The check and the write are one statement, so there is no gap between them.
- Transactions use PostgreSQL's default isolation (read committed). A design doc may ask for stronger isolation for a specific operation, such as booking a production slot that only one job may take, and says why.
- Races on uniqueness, such as two people creating a customer with the same email address, are caught by unique constraints and returned as `409 Conflict` with code `duplicate`.

### The background worker

The worker follows the same rule. It reads the object, makes its change with the version it read, and on a conflict reads again and retries a small number of times. A worker change that conflicts with a user's change never overwrites it.

### In the UI

- When the UI loads an object it keeps its ETag, and sends it with every change.
- **On `412`, the UI fetches the latest version and compares.**
    - If the other change touched none of the fields this user changed, the UI reapplies this user's changes on top of the latest version automatically and tells them quietly ("Updated. Alex also changed the due date.").
    - If both changed the same field, or the action depends on the whole object (sending, approving, invoicing), the UI stops and shows what changed and who changed it, using the audit trail's snapshots ([ADR 0019](0019-audit-trail.md)). The user chooses to keep their version, take the other one, or start again.
- **Kanban moves** are actions with `If-Match`. If a card has moved or changed since the board loaded, the move is refused, the card snaps to its current place and the board refreshes it.
- Live updates (the events and live updates ADR) will make conflicts rarer by refreshing what is on screen as it changes. They do not replace these checks.

## Options considered

1. **Optimistic concurrency with an aggregate version, ETags and required `If-Match`**: no locks to manage or clear up, works for the web app and integrations alike, standard HTTP, and conflicts are visible. **Chosen.**
2. **Last write wins**: no work at all; changes are lost silently, which is unacceptable for prices and invoices.
3. **Pessimistic locking** ("Alex is editing this quote"): no conflicts while the lock holds; locks are left behind when tabs close or connections drop, need timeouts and overrides, and block integrations. A presence indicator without a lock may come with live updates.
4. **Optional `If-Match`**: friendlier to simple integrations; clients that leave it out get last-write-wins without anyone noticing, including our own code if someone forgets.
5. **Field-level merging on the server**: fewer conflicts; the server cannot know which changes depend on each other, and the rules become complicated. The UI's automatic reapply covers the common case.
6. **PostgreSQL `xmin` as the token**: no column to maintain; covers only one row, and changes on dump and restore.

## Consequences

- Every aggregate root table has a `concurrency_version` column, and a schema test fails if one is missing, like the `tenant_id` test ([ADR 0012](0012-testing.md)).
- The API reference explains `ETag`, `If-Match` and `If-Match: *`, and the generated client sends `If-Match` automatically from the object it last read.
- Every endpoint that changes an object has tests for a missing `If-Match` (428), a stale one (412) and a correct one.
- The UI needs a conflict dialog and the comparison logic. It is designed once in the front-end design doc and used by every form.
- Integrations must read before they write, or use `If-Match: *` knowingly.

## Questions for review

1. **Actions need `If-Match`** as well as `PATCH` and `DELETE`. This protects against sending or approving a quote that changed a minute ago, but every action call needs the ETag. Agreed?
2. **`If-Match: *` is allowed** for integrations that want last-write-wins, recorded in the audit trail. Or should it be refused, forcing every client to read first?
3. **The UI reapplies non-overlapping changes automatically** and only asks when the same field was changed, or the action depends on the whole object. Or should it always ask?
4. **`etag` appears in the JSON** as well as the header, so grids can edit items from a collection. Agreed?
5. **`If-None-Match` and `304`** on single-object reads from the start, or later when there is a need?
6. **A presence indicator** ("Alex is viewing this quote") with live updates later, without locks. Worth keeping on the list?
