# 0017. Object IDs

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Every object in Signboard (a user, an account, a quote, a job, an invoice line) needs an identifier that the database, the API, integrations, logs and people all use. Signboard is multi-tenant, with every business's records in shared tables ([ADR 0004](0004-multi-tenant.md)), and has one public API for every actor ([ADR 0010](0010-api-first.md)).

That rules out the usual default, an auto-increment integer: in shared tables it would show every business how many records everyone else has created, and it makes records easy to enumerate. Random UUIDs reveal nothing, but they are long, cannot be read out over the phone, and do not say what kind of object they identify.

People will read IDs to each other: on the phone to a customer, in a support email, from a job ticket on the shop floor. A type prefix makes an ID self-explanatory in a log or an error message.

This record is about the system's own identifier for each object. Other identifiers that particular actors use for an object (a customer's purchase order number, a number carried over from a previous system, an accounting system's reference) are a separate, later decision.

## Decision

Every object has exactly **one** ID, assigned by Signboard when the object is created. It is the object's primary key in the database and its identifier in the API, in foreign keys, in logs and on screen. There is no second, internal key.

**Format:** a three-letter type prefix and random digits in groups of four:

```
ACC-4821-0937
JOB-4821-0937-1156
```

- **Prefix:** three uppercase letters identifying the object type, such as `JOB` for a job or `QUO` for a quote. Each type has exactly one prefix and each prefix belongs to exactly one type. Prefixes are listed in the [data model](../design/data-model.md) and never change once in use.
- **Digits:** decimal digits from a cryptographically secure random number generator, uniformly distributed, leading zeros allowed. They encode nothing: no business, no date, no sequence.
- **Length per type:** the number of digits is a multiple of four, chosen for each object type when its requirements are written, and fixed for that type once in use. The space must comfortably exceed the type's expected volume: at ten times the expected lifetime count, fewer than one insert in 10,000 should collide. Rarely created types such as accounts may need eight digits; high-volume types such as jobs or quote lines twelve or more.
- **Canonical form:** uppercase, with hyphens. The database stores the canonical form, the API returns it and accepts only it. Screens that let people type or paste an ID tidy what was entered (case, spaces, missing hyphens) before sending it.

**Uniqueness, checked at generation:**

- An ID is unique across every business, because all businesses share the same tables.
- The primary key's unique constraint is the check. A new ID is generated and the insert attempted. If it collides, a fresh ID is generated and the insert retried, up to five times, after which the request fails with an error. A lookup before inserting is not used: it would race with concurrent inserts, and row-level security would hide other businesses' rows from it, so it would miss exactly the collisions that matter.
- The chance that an insert collides is the number of existing objects of that type divided by the number of possible IDs. With twelve digits (10<sup>12</sup> IDs), a type with ten million objects collides on about one insert in every 100,000.

**Validation:**

- An ID with the wrong prefix for the endpoint (`GET /api/v1/jobs/QUO-…`) is rejected without touching the database: `404 Not Found` in a path, `400 Bad Request` in a request body.
- The database enforces the format with a check constraint on every ID column, including the prefix and the length for that table.

**IDs never change and are never reissued.** An object keeps its ID for life. Rows are soft-deleted by default ([ADR 0018](0018-soft-delete.md)), so a deleted object's row, and its ID, stay in the table, and the primary key constraint keeps the ID from being issued again. For the rare object types that are hard-deleted, an ID could in theory be issued again after deletion; that is accepted.

**Versions are not objects.** Where an object has versions, it keeps one ID for its whole life and its versions are numbered beneath it. For example, a quote is always `QUO-…`, and its revisions are versions 1, 2, 3 of that quote.

## Options considered

1. **Prefixed random digits, collision-checked, one ID per object, length per type**: readable aloud, self-describing, reveals nothing, one identifier everywhere, no longer than each type needs. Needs a retry on collision, which is rare. **Chosen.**
2. **Auto-increment integers**: compact and simple; reveal volumes across businesses in shared tables and invite enumeration.
3. **UUID (version 4 or 7)**: no collision handling needed; 36 characters, cannot be read aloud, carries no type. Version 7 also reveals the creation time.
4. **Stripe-style prefixed encoded IDs** (`job_01JA8Z3K7Q9X2M4N6P8R0T`): typed and collision-free by construction; long and awkward to say.
5. **A UUID primary key behind a public prefixed ID**: lets the public format change without touching foreign keys; doubles the identifiers in every table, log and debugging session, as insurance against a change there is no reason to expect.
6. **Store the digits as a 64-bit integer and add the prefix at the edges**: smaller indexes; the value in the database is no longer the value people see, so an ID from a log cannot be pasted into a query. Not worth it at Signboard's scale.

## Consequences

- One identifier everywhere: an ID copied from an email, a log or the API can be pasted straight into a database query or a URL.
- Primary keys are short text (13 characters for eight digits, 18 for twelve) rather than integers or UUIDs. Indexes are somewhat larger, which is acceptable at Signboard's data volumes.
- IDs are not time-ordered. Lists sort by explicit fields such as the creation time, with the ID as the final, stable tie-breaker ([ADR 0016](0016-rql-queries.md)).
- ID generation and the retry on collision live in one shared component used by every insert, and are tested, including forced collisions ([ADR 0012](0012-testing.md)).
- Adding an object type means registering its prefix and digit length in the data model, as part of its requirements.
- Numbers that people and other systems use for an object, such as a previous system's quote numbers that customers still quote back, sequential invoice numbers, and the numbering formats listed in [ADR 0006](0006-configuration-over-customisation.md), are not object IDs. They wait for the later decision on additional, actor-scoped identifiers. The [migration PRD](../requirements/prd-migration.md) depends on that decision.
