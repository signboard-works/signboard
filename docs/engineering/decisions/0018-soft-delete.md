# 0018. Soft deletion by default

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

A sign shop's records reference each other for years: an invoice points at a sales order, which points at a quote, a customer and the user who sent it. If a row is physically deleted, everything that referenced it either breaks or must be deleted too, and the history of what happened is lost. People also delete things by mistake and want them back.

Object IDs must never be reissued ([ADR 0017](0017-object-ids.md)). That is only guaranteed while the row holding an ID still exists.

## Decision

Rows are **soft-deleted by default**: deleting an object marks it as deleted rather than removing it.

- Every table has `deleted_at` (a timestamp, empty while the object is live) and `deleted_by` (the actor who deleted it). Deletion is also recorded in the audit log.
- Deleted objects are excluded from every read by default, by an EF Core global query filter applied alongside the tenant filter ([ADR 0004](0004-multi-tenant.md)). Fetching a deleted object by ID returns `404 Not Found`, unless a resource's design says otherwise.
- Deleted objects can still be referenced. An invoice for a customer who has since been deleted still shows that customer.
- Whether a deleted object can be restored, and by whom, is decided in each resource's design.
- Uniqueness rules that apply only to live objects (for example, a name unique within a business) use partial unique indexes that ignore deleted rows. Object IDs are unique across deleted rows too.

**Hard deletion is the exception**, and needs a reason in the design doc. The expected exceptions are:

1. **A business leaving the hosted service:** its data can be deleted completely ([ADR 0004](0004-multi-tenant.md)).
2. **Personal information that must be destroyed:** under the Australian Privacy Act, personal information that is no longer needed must be destroyed or de-identified. The default is **de-identification**: personal fields are cleared or replaced and the row, its ID and its references remain. A row is physically removed only where de-identification is not enough.
3. **Short-lived technical data** with no historical value, such as sessions, expired tokens and caches.

## Options considered

1. **Soft deletion by default, hard deletion by justified exception**: history and references stay intact, mistakes can be undone, IDs are never reissued. Every query must exclude deleted rows, and deleted data is still held. **Chosen.**
2. **Hard deletion with an audit log or history tables**: tables hold only live data; the history is kept elsewhere, but references from live records to deleted ones break.
3. **Move deleted rows to archive tables**: live tables stay clean; every table needs an archive twin, and references must follow the row between tables.

## Consequences

- Excluding deleted rows is central (query filters), never left to individual queries. Every endpoint's tests check that deleted objects are not returned ([ADR 0012](0012-testing.md)).
- Tables only grow. A retention policy for long-deleted rows may be needed later and would be its own decision.
- Deleted data is still personal data while it is held, so deletion requests under the Privacy Act use de-identification, not soft deletion alone.
- Indexes on large tables may need to be partial (live rows only) to stay efficient.
