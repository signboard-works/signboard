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
| JSON types | Amounts, rates and quantities are plain numbers (never strings), percentages are percent (`10` is 10%) in fields ending `Percent`. Moments are UTC (`2026-10-11T03:15:00Z`) in fields ending `At`; dates are `2026-10-11` in fields ending `Date`. Every field you can see is present: `null` means no value, a missing field means it isn't available to you | [ADR 0029](../engineering/decisions/0029-api-errors-idempotency-versioning.md#json-types) |
| Money and tax | One `currency` per document. Tax is calculated and rounded per line, half away from zero; totals are sums of rounded lines. Read amounts into a decimal type and never add them up in floating point | [ADR 0036](../engineering/decisions/0036-money-tax-and-time.md) |
| Languages | The API returns codes, never display labels. Error messages follow `Accept-Language`, and the response says which language it used in `Content-Language` | [ADR 0037](../engineering/decisions/0037-localisation.md) |
| Rate limits | Fair-use limits per user, per API token and per business. Over a limit you get `429 rate_limited` with `Retry-After`; slow down and retry. Use bulk and import endpoints rather than thousands of single requests | [ADR 0038](../engineering/decisions/0038-rate-limiting.md) |
| Security | TLS 1.3 only. No CORS: call the API from your server with an API token, never from another website's scripts. Never put a token in a URL | [ADR 0039](../engineering/decisions/0039-security-baseline.md) |
| Errors | Problem details with a stable `code`, a `traceId` and field errors. Clients branch on `code`, never on wording | [ADR 0029](../engineering/decisions/0029-api-errors-idempotency-versioning.md) |
| Concurrency | Every object has a `revision`, returned as the `ETag`. `PATCH`, `DELETE` and actions must send `If-Match`; a stale one gets `412` | [ADR 0031](../engineering/decisions/0031-concurrency.md) |
| Live updates | `GET /api/v1/system/events/stream` is a Server-Sent Events stream of thin notifications ("this object changed") for objects you can see. Fetch the object to get its data | [ADR 0032](../engineering/decisions/0032-events-webhooks-live-updates.md) |
| Webhooks | Signed with the Standard Webhooks specification; the payload includes the object as the webhook's owner sees it. Deduplicate on `webhook-id`, order by `revision` | [ADR 0032](../engineering/decisions/0032-events-webhooks-live-updates.md) |
| Unreleased features | Features behind a feature flag don't exist for you until released: `404`, no fields, not in this reference. `GET /api/v1/system/features` lists the flags that are on for you | [ADR 0033](../engineering/decisions/0033-feature-flags.md) |
| Retries | Send an `Idempotency-Key` header on every `POST`; a retry with the same key returns the original response. Retry network errors, `429` and `503` with backoff, honouring `Retry-After`; never retry other `4xx` | [ADR 0029](../engineering/decisions/0029-api-errors-idempotency-versioning.md) |
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
