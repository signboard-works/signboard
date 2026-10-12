# 0045. Search

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-12 |
| **Deciders** | Stuart Meeks |

## Context

People at a sign shop look things up constantly: a customer rings about "the banner for the school fete", someone reads a PO number off an email, a job number is scribbled on a job sheet. They need one box that finds anything they are allowed to see: quotes, sales orders, jobs, invoices and customers, by object ID, name, reference, PO number, address or a word from a description.

Grids already filter, sort and page through RQL ([ADR 0016](0016-rql-queries.md)), with per-property allow-lists. Search is different: one query across many kinds of object, tolerant of typos and partial words, ranked by relevance.

Search must respect everything already decided:

- **Tenancy and visibility.** Row-level security applies to every query ([ADR 0004](0004-multi-tenant.md)), and hidden fields can never be selected, filtered or sorted ([ADR 0010](0010-api-first.md), [ADR 0016](0016-rql-queries.md)). Search must not become a way round either: a customer user must not find a quote by searching for a cost price.
- **No extra services.** Background work avoided Redis and brokers so that self-hosters need only PostgreSQL ([ADR 0034](0034-background-work.md)), and initial development is kept as cheap as possible.
- **Locale.** [ADR 0037](0037-localisation.md) left sorting and searching text by locale to this ADR.

## Decision

### Search runs in PostgreSQL

- Search uses PostgreSQL's built-in **full-text search** (`tsvector` columns with GIN indexes) for words and phrases, and the **`pg_trgm`** extension for partial words, typos and prefixes, such as "fet" finding "fete" or "Smtih" finding "Smith". Both are available on RDS for PostgreSQL 18, alongside `unaccent` ([RDS extensions](https://docs.aws.amazon.com/AmazonRDS/latest/PostgreSQLReleaseNotes/postgresql-extensions.html)).
- There is **no separate search engine**. Nothing new to run, pay for, secure, back up or install when self-hosting.
- The search code sits behind a Signboard interface in the `system` module, so a dedicated engine could replace it later without touching callers ([ADR 0026](0026-modular-monolith.md)).

### What is searchable

- **Search documents per object and field group.** Each searchable object type declares, in code, which of its fields are searchable and with what weight, such as the ID and number highest, then names and references, then descriptions and addresses. Every field already belongs to a field group ([ADR 0022](0022-authorisation-and-roles.md)), so an object has one search document for each field group with searchable fields in it, such as its general fields, its pricing and its internal notes.
- **A search looks only at the field groups the actor may see.** Production staff can find a job by its description but not by its cost price; staff can search internal notes; a customer user searches only what a customer may see. Nobody can find an object by a field they could not read. A test checks every searchable field group against every role, including customer roles.
- **Excerpts come from visible field groups only.** The short excerpt shown with a result is built only from documents the actor was allowed to search.
- **Search documents carry `tenant_id`** and have row-level security like every tenant-owned table, so results never cross businesses.
- **Results are filtered by the same policy layer as the API** ([ADR 0010](0010-api-first.md)): a customer user only gets objects belonging to their customer account, and never sees an object they could not open.

### Keeping it current

- Search documents live in a `search_documents` table owned by the `system` module, with one row per object and field group.
- **An object's own fields are indexed in the same transaction** as the change, by the application's save path: the owning module produces the object's search text through the search interface, and `system` writes it ([ADR 0026](0026-modular-monolith.md)). Search text is not built in database triggers, which would put every object type's rules in SQL and have one module reading every other module's tables. An object is searchable the moment it is saved, and never searchable after it is deleted ([ADR 0018](0018-soft-delete.md)).
- **Text copied from related objects**, such as the customer's name on each of their jobs, is updated asynchronously: the change event from the outbox ([ADR 0032](0032-events-webhooks-live-updates.md)) starts a job that re-indexes the related objects, usually within seconds. Renaming a customer does not rewrite hundreds of rows inside one transaction.
- A rebuild job re-creates every search document for a business, used after a change to what is searchable or a restore ([ADR 0034](0034-background-work.md), [ADR 0043](0043-backups-and-recovery.md)).

### Object IDs and order numbers jump straight to the object

If the query is a complete object ID, such as `JOB-4821-0937-1156`, in any case and with or without hyphens ([ADR 0017](0017-object-ids.md)), and the caller can see that object, search returns it as the only result and the web app opens it directly. The same applies to a customer's purchase order number that matches exactly one object the caller can see. A partial ID, such as `JOB-4821`, matches by prefix.

### Language, case and accents

- Matching ignores **case and accents**: "cafe" finds "Café".
- **Names, references and numbers** are indexed with the `simple` configuration and `unaccent`, without word stemming, because stemming mangles them.
- **Descriptions and notes** are indexed twice: unstemmed, and with English stemming, so "printing" also finds "printed". A field's type in the searchable-field declaration says which it is. Other languages' stemming is added when there are businesses using them ([ADR 0037](0037-localisation.md)).
- **Sorting text by locale** in grids uses ICU collations, which RDS for PostgreSQL includes ([RDS collations](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL-Collations.html)): sortable text columns use a case- and accent-insensitive ICU collation based on the root locale (`und`), so "émile", "Emile" and "emile" sort together and in natural order. PostgreSQL 18 allows `LIKE` with such nondeterministic collations ([PostgreSQL 18 release notes](https://www.postgresql.org/docs/18/release-18.html)), so RQL's text filters work on them. A per-business collation is not needed while every business sorts in English.

### The API

- **`GET /api/v1/system/search?q=…`** returns a ranked list of typed results: each with the object's ID, type, title, a short excerpt and its URL ([ADR 0020](0020-url-structure.md)). Results never contain fields the caller cannot see. Responses are `Cache-Control: no-store`, so no cache can serve one actor's results to another.
- `type` narrows the results to object types, such as `type=quote,job`. Results are capped at 50, with no paging: search is for finding, and grids are for browsing. The web app offers **"See all"** for each type, which opens that collection's grid with the same text filter, where paging works as it does everywhere else.
- Invalid queries return problem details ([ADR 0029](0029-api-errors-idempotency-versioning.md)). Search counts towards fair-use rate limits like any request ([ADR 0038](0038-rate-limiting.md)).
- **Grids** keep RQL for filtering. Each collection may also offer a text filter on its allow-listed text fields, using the same matching rules; whether Mpt.Rql can express it directly or needs a custom operator is checked when the first grid is built.

### Performance and when to revisit

- **Target:** results in under 200 ms at the 95th percentile for a business with 100,000 searchable objects.
- We revisit this decision, and consider a dedicated engine behind the same interface, if search misses that target with sensible indexes, or needs features PostgreSQL cannot provide, such as typo tolerance across very large text or rich facets.

## Options considered

### Search engine

1. **PostgreSQL full-text search and `pg_trgm`**: nothing new to run or pay for; results are transactionally current; row-level security applies; works the same self-hosted. Weaker relevance ranking and typo tolerance than a dedicated engine. **Chosen.**
2. **A dedicated engine** (OpenSearch, Meilisearch, Typesense): better relevance, typo tolerance and facets; another service to run, pay for, secure and keep in step, with its own copy of every business's data outside row-level security, and another thing for self-hosters to install. A managed service such as Amazon OpenSearch Service would be a separate monthly cost on top of the database.
3. **RQL text filters only, no global search**: no new work; nobody can find a job from a half-remembered word without knowing which list to look in.

### Hiding fields from search

1. **One document per field group, searched only where the actor may see the group**: follows the visibility rules already decided, so staff can search notes and nobody can search what they cannot read. **Chosen.**
2. **One shared document plus a staff-only one**: simpler; staff roles differ, so a production user could find a job by its price.
3. **Only fields everyone can see**: simplest and safest; staff could not search their own notes.

### Keeping the index current

1. **Own fields in the same transaction, related objects' text by a job**: what was just saved is always findable, and a rename does not hold a long transaction. **Chosen.**
2. **Everything in the same transaction, by database triggers**: never stale; search rules in SQL, cross-module reads, and large transactions for renames.
3. **Everything asynchronously from the outbox**: simplest writes; what was just saved may not be findable for a few seconds.

## Consequences

- Each searchable object type declares its searchable fields, their weights and whether they are stemmed, in code, and its design doc lists them with their field groups.
- The visibility test for search documents runs alongside the field visibility tests ([ADR 0012](0012-testing.md)), for every field group and role, including a cross-tenant case and a customer-user case.
- The `pg_trgm` and `unaccent` extensions are enabled by migrations, and are available on standard PostgreSQL for self-hosters.
- The ICU collation's behaviour, including `LIKE` support, is confirmed against RDS for PostgreSQL 18 in the first data-model work.
- The web app gets one search box, available everywhere, with keyboard access and announced results ([ADR 0042](0042-accessibility.md)).
- No separate search service cost is added. The search table and its indexes use some database storage and work, within the database's current size.
