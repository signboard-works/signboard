# 0045. Search

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

> **Draft for review.** Written from Claude's recommendations. The choices still open are listed under [Questions for review](#questions-for-review).

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

- **A search document per object.** Each searchable object type declares, in code, which of its fields are searchable and with what weight, such as the ID and number highest, then names and references, then descriptions and addresses.
- **Only fields that are visible to every actor who can see the object are put in the shared search document.** Fields visible to some actors only, such as cost prices or internal notes, are either left out or held in a separate, staff-only document that is searched only for actors who may see those fields. A test fails if a searchable field is not visible to every actor the document is searched for.
- **Search documents carry `tenant_id`** and have row-level security like every tenant-owned table, so results never cross businesses.
- **Results are filtered by the same policy layer as the API** ([ADR 0010](0010-api-first.md)): a customer user only gets objects belonging to their customer account, and never sees an object they could not open.

### Keeping it current

- Search documents live in a `search_documents` table owned by the `system` module, one row per searchable object, updated **in the same transaction** as the change, by the same mechanism that writes change events to the outbox from the revision trigger ([ADR 0031](0031-concurrency.md), [ADR 0032](0032-events-webhooks-live-updates.md)). An object is searchable the moment it is saved, and never searchable after it is deleted ([ADR 0018](0018-soft-delete.md)).
- A rebuild job re-creates every search document for a business, used after a change to what is searchable or a restore ([ADR 0034](0034-background-work.md), [ADR 0043](0043-backups-recovery-and-migrations.md)).

### Object IDs jump straight to the object

If the query is a complete object ID, such as `JOB-4821-0937-1156`, in any case and with or without hyphens ([ADR 0017](0017-object-ids.md)), and the caller can see that object, search returns it as the only result and the web app opens it directly. A partial ID, such as `JOB-4821`, matches by prefix.

### Language, case and accents

- Matching ignores **case and accents**: "cafe" finds "Café". Full-text search uses the `simple` configuration with `unaccent`, rather than a language's word stemming, because most searched text is names, references and numbers, and stemming mangles them. Stemming can be added per language when there are languages other than English ([ADR 0037](0037-localisation.md)).
- **Sorting text by locale** in grids uses ICU collations, which RDS for PostgreSQL includes ([RDS collations](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL-Collations.html)): sortable text columns use a case- and accent-insensitive ICU collation based on the root locale (`und`), so "émile", "Emile" and "emile" sort together and in natural order. PostgreSQL 18 allows `LIKE` with such nondeterministic collations ([PostgreSQL 18 release notes](https://www.postgresql.org/docs/18/release-18.html)), so RQL's text filters work on them. A per-business collation is not needed while every business sorts in English.

### The API

- **`GET /api/v1/system/search?q=…`** returns a ranked list of typed results: each with the object's ID, type, title, a short excerpt and its URL ([ADR 0020](0020-url-structure.md)). Results never contain fields the caller cannot see.
- `type` narrows the results to object types, such as `type=quote,job`. Results are capped at 50, with no deep paging: search is for finding, and grids are for browsing.
- Invalid queries return problem details ([ADR 0029](0029-api-errors-idempotency-versioning.md)). Search counts towards fair-use rate limits like any request ([ADR 0038](0038-rate-limiting.md)).
- **Grids** keep RQL for filtering. Each collection may also offer a text filter on its allow-listed text fields, using the same matching rules; whether Mpt.Rql can express it directly or needs a custom operator is checked when the first grid is built.

### Performance and when to revisit

- **Target:** results in under 200 ms at the 95th percentile for a business with 100,000 searchable objects.
- We revisit this decision, and consider a dedicated engine behind the same interface, if search misses that target with sensible indexes, or needs features PostgreSQL cannot provide, such as typo tolerance across very large text or rich facets.

## Options considered

1. **PostgreSQL full-text search and `pg_trgm`**: nothing new to run or pay for; results are transactionally current; row-level security applies; works the same self-hosted. Weaker relevance ranking and typo tolerance than a dedicated engine. **Chosen.**
2. **A dedicated engine** (OpenSearch, Meilisearch, Typesense): better relevance, typo tolerance and facets; another service to run, pay for, secure and keep in step, with its own copy of every business's data outside row-level security, and another thing for self-hosters to install. Amazon OpenSearch Service alone would cost more each month than the database.
3. **RQL text filters only, no global search**: no new work; nobody can find a job from a half-remembered word without knowing which list to look in.

## Consequences

- Each searchable object type declares its searchable fields and weights in code, and its design doc lists them, with which actors may search each one.
- The visibility test for search documents runs alongside the field visibility tests ([ADR 0012](0012-testing.md)), including a cross-tenant case and a customer-user case.
- The `pg_trgm` and `unaccent` extensions are enabled by migrations, and are available on standard PostgreSQL for self-hosters.
- The ICU collation's behaviour, including `LIKE` support, is confirmed against RDS for PostgreSQL 18 in the first data-model work.
- The web app gets one search box, available everywhere, with keyboard access and announced results ([ADR 0042](0042-accessibility.md)).
- No cost is added.

## Questions for review

1. **PostgreSQL only**, no dedicated search engine, until it misses its target. Or plan for Meilisearch or Typesense from the start?
2. **A search table updated in the same transaction**, so results are never stale. Or update it asynchronously from the outbox, which is simpler for writes but can lag by seconds?
3. **A separate staff-only search document** for fields customers cannot see, such as internal notes. Or keep staff-only fields out of search altogether, which is simpler and safer?
4. **No word stemming**, so "printing" does not find "printed". Better for names and references; worse for descriptions. Or English stemming from the start?
5. **One case- and accent-insensitive ICU collation for sorting**, for every business. Or a collation per business locale now?
6. **A typed object ID jumps straight to the object.** Agreed?
7. **Results capped at 50 with no paging.** Or page through all matches?
8. **The 200 ms target at 100,000 objects per business.** Too strict, or too loose?
