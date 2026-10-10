# 0021. Every object request returns the full object

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

APIs often answer writes with little or nothing: `204 No Content` for a delete, just an ID for a create, only the changed fields for an update. Clients then make a second request to see the result, and may see a different state if something else changed in between: a computed price, a status moved on by an action, a field set by a rule. Signboard's own app is an ordinary API client ([ADR 0010](0010-api-first.md)), so every extra round trip is paid by its users too.

## Decision

Every request on a single object returns **the full object** as its response body, in its state after the request:

| Request | Status | Body |
|---|---|---|
| `GET` an object | `200 OK` | The object |
| `POST` to a collection (create) | `201 Created` | The new object, including its ID ([ADR 0017](0017-object-ids.md)) |
| `PATCH` an object | `200 OK` | The object after the update |
| `POST` an action (`/send`, `/approve`, …) | `200 OK` | The object after the action |
| `DELETE` an object | `200 OK` | The soft-deleted object, with `deletedAt` and `deletedBy` set ([ADR 0018](0018-soft-delete.md)) |

Signboard has no `PUT` ([ADR 0020](0020-url-structure.md)); if one were ever added, the same rule would apply.

- **Full means the whole representation**, not only the fields that changed or were sent. An object's representation, including which dependent child objects it embeds, is defined in that object's design.
- **Field visibility still applies** ([ADR 0010](0010-api-first.md)). The response omits every field the actor may not see, exactly as a `GET` would. Writing an object never reveals more of it.
- The response is the same representation a `GET` immediately afterwards would return, so a client never needs to re-fetch after a write.
- Collection requests return objects inside the paging envelope, and RQL `select` may narrow them ([ADR 0016](0016-rql-queries.md)). This record covers requests on a single object.

## Options considered

1. **Full object on every request**: one representation for every operation; clients always see the result, including computed and rule-set fields; no re-fetch. Larger responses. **Chosen.**
2. **Minimal responses** (`204 No Content`, an ID, or only changed fields): smaller; every client must re-fetch to see the result, and may see a later state than its own write produced.
3. **Optional full response** (for example via a `Prefer: return=representation` header): flexible; two behaviours to document and test for every endpoint.

## Consequences

- Every write endpoint ends by loading and serialising the object through the same path, and the same field-visibility policy, as `GET`. Tests check that a write's response equals a subsequent `GET` for each actor ([ADR 0012](0012-testing.md)).
- Responses to writes are larger, which is acceptable for a back-office API.
- Objects that embed large child collections need a deliberate representation in their design, because the rule applies to every write on them.
