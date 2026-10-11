# API reference

Signboard has one public API, used by its own web app, by customers and by integrations alike. There are no private endpoints ([ADR 0010](../engineering/decisions/0010-api-first.md)).

The reference for each module will be generated from its OpenAPI contract, which is written in the module's design document before any code. Until then, this page summarises the conventions every endpoint follows.

## Conventions

| Topic | Convention | Decided in |
|---|---|---|
| Paths | `/api/v1/{module}/{collection}/{id}`, with modules `accounts`, `catalog`, `sales`, `production`, `billing`, `notifications` and `system`. No account in the path | [ADR 0020](../engineering/decisions/0020-url-structure.md) |
| Methods | `GET`, `POST`, `PATCH` (JSON Merge Patch) and `DELETE`; no `PUT`. Actions are `POST` to a verb sub-path, such as `/quotes/{id}/send` | [ADR 0020](../engineering/decisions/0020-url-structure.md) |
| Responses | Every request on an object returns the full object, with fields hidden from the caller omitted | [ADR 0020](../engineering/decisions/0020-url-structure.md) |
| Object IDs | One ID per object, such as `JOB-4821-0937-1156`: a three-letter prefix and random digits in groups of four | [ADR 0017](../engineering/decisions/0017-object-ids.md) |
| Queries | Collections take RQL for filtering, sorting, selecting and paging. `limit` defaults to 50, with a maximum of 1000, and the total is always returned | [ADR 0016](../engineering/decisions/0016-rql-queries.md) |
| Authentication | Cookie sessions in the browser; hashed, scoped, expiring API tokens for integrations | [ADR 0021](../engineering/decisions/0021-authentication.md) |
| Authorisation | Each request acts as one membership. Permissions are checked per request and deny by default; fields are visible per actor | [ADR 0022](../engineering/decisions/0022-authorisation-and-roles.md) |
| Deletion | Deleting an object soft-deletes it | [ADR 0018](../engineering/decisions/0018-soft-delete.md) |
| Errors | Problem details with a stable `code`, a `traceId` and field errors. Clients branch on `code`, never on wording | [ADR 0029](../engineering/decisions/0029-api-errors-idempotency-versioning.md) |
| Concurrency | Every object has a `revision`, returned as the `ETag`. `PATCH`, `DELETE` and actions must send `If-Match`; a stale one gets `412` | [ADR 0031](../engineering/decisions/0031-concurrency.md) |
| Live updates | `GET /api/v1/system/events/stream` is a Server-Sent Events stream of thin notifications ("this object changed") for objects you can see. Fetch the object to get its data | [ADR 0032](../engineering/decisions/0032-events-webhooks-live-updates.md) |
| Webhooks | Signed with the Standard Webhooks specification; the payload includes the object as the webhook's owner sees it. Deduplicate on `webhook-id`, order by `revision` | [ADR 0032](../engineering/decisions/0032-events-webhooks-live-updates.md) |
| Unreleased features | Features behind a feature flag don't exist for you until released: `404`, no fields, not in this reference. `GET /api/v1/system/features` lists the flags that are on for you | [ADR 0033](../engineering/decisions/0033-feature-flags.md) |
| Retries | Send an `Idempotency-Key` header on every `POST`; a retry with the same key returns the original response | [ADR 0029](../engineering/decisions/0029-api-errors-idempotency-versioning.md) |
| Versioning | Additive changes happen within `v1`, so clients must ignore unknown fields and handle unknown enumeration values and error codes | [ADR 0029](../engineering/decisions/0029-api-errors-idempotency-versioning.md) |

A collection response looks like this:

```json
{
  "$meta": {
    "pagination": { "offset": 0, "limit": 50, "total": 128 }
  },
  "data": []
}
```

An error looks like this:

```json
{
  "type": "https://docs.signboard.works/api/errors/validation-failed",
  "title": "The request has invalid fields.",
  "status": 422,
  "code": "validation_failed",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [
    { "pointer": "/lines/2/quantity", "code": "must_be_positive", "message": "Quantity must be greater than zero." }
  ]
}
```

A page for each error code will be added here with the first endpoints.
