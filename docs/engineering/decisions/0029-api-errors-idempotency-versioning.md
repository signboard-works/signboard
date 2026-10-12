# 0029. API errors, idempotency and versioning

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

## Context

Signboard has one public API for its own web app, customers and integrations ([ADR 0010](0010-api-first.md)). Three things about that API affect every endpoint and every client, so they are decided once, before the first endpoint is written:

- **Errors.** Clients need to tell what went wrong without parsing English, show the right message against the right field, and give support something to search for. ASP.NET Core's default error output is problem details with a generic `type` that links to the HTTP standard's definition of the status code, a dictionary of field messages, and the parser's own wording. It is a sensible default, but it does not identify the error.
- **Retries.** A network failure part-way through a `POST` leaves the client unsure whether it worked. Retrying blindly can create a second invoice or send a quote twice.
- **JSON types.** JSON has few types, and clients in different languages read them differently. Every field must mean the same thing to every client.
- **Change.** The API will change as Signboard grows. Integrators need to know which changes can break them, and our own UI and API are deployed separately ([ADR 0015](0015-separate-deployments-one-origin.md)), so for a short time a UI build talks to an API one release newer.

## Decision

### Errors

Every error response is **problem details** ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)) with the media type `application/problem+json`. We replace ASP.NET Core's default problem details output in one place, so errors raised by the framework (such as unreadable JSON) come out in the same shape as our own.

```json
{
  "type": "https://docs.signboard.works/api/errors/validation-failed",
  "title": "The request has invalid fields.",
  "status": 422,
  "code": "validation_failed",
  "detail": "2 fields are invalid.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [
    { "pointer": "/lines/2/quantity", "code": "must_be_positive", "message": "Quantity must be greater than zero." },
    { "pointer": "/customerId", "code": "required", "message": "A customer is required." }
  ]
}
```

| Member | Meaning |
|---|---|
| `type` | A link to the error's page in the API reference. Every `code` has one |
| `title` | A short, human-readable summary of the error type. Not for parsing |
| `status` | The HTTP status code, repeated for convenience |
| `code` | The stable, machine-readable error code, in `snake_case`. Clients branch on this |
| `detail` | Optional. A human-readable explanation of this occurrence. Not for parsing |
| `traceId` | The request's trace ID, which is also the audit correlation ID ([ADR 0025](0025-observability.md)). Support can search logs and the audit trail by it |
| `errors` | For validation errors: one entry per problem, each with a JSON Pointer ([RFC 6901](https://www.rfc-editor.org/rfc/rfc6901)) to the field in the request body, a `code` and a `message` |

Rules:

- Clients rely on `code`, the field `code`s and `status`. The wording of `title`, `detail` and `message` may change, and may be translated later ([ADR 0037](0037-localisation.md)).
- An error code, once published, keeps its meaning. New codes may be added at any time ([versioning](#versioning)).
- Error responses never contain stack traces, SQL, internal identifiers other than object IDs, or anything from another business. Server errors give a generic message and the `traceId`.

### Status codes

| Status | When | Example codes |
|---|---|---|
| `400 Bad Request` | The request cannot be understood: unreadable JSON, a wrong content type in the body, an invalid RQL query ([ADR 0016](0016-rql-queries.md)), or an unknown field | `malformed_request`, `invalid_query`, `unknown_field` |
| `401 Unauthorized` | No valid session or token | `unauthenticated` |
| `403 Forbidden` | The caller lacks a permission ([ADR 0022](0022-authorisation-and-roles.md)) | `forbidden` |
| `403 Forbidden` | The business's account is read-only, because it has not paid or is closing ([ADR 0047](0047-sign-up-plans-and-billing.md#subscription-status)) | `account_read_only` |
| `403 Forbidden` | The business's plan does not include the feature ([ADR 0046](0046-entitlements.md)) | `not_entitled` |
| `403 Forbidden` | The request would go beyond one of the plan's limits, with the limit, its maximum and current usage ([ADR 0046](0046-entitlements.md#limits)) | `limit_reached` |
| `404 Not Found` | The object does not exist, is deleted ([ADR 0018](0018-soft-delete.md)), belongs to another business, or is not visible to the caller | `not_found` |
| `409 Conflict` | The object's state does not allow the request, such as sending a quote that has already been accepted; or a retry arrives while the original request is still running | `invalid_state`, `request_in_progress` |
| `412 Precondition Failed`, `428 Precondition Required` | Concurrency checks ([ADR 0031](0031-concurrency.md)) | `precondition_failed`, `precondition_required` |
| `422 Unprocessable Content` | The request is well-formed but its values are invalid; or an idempotency key is reused for a different request | `validation_failed`, `idempotency_key_reused` |
| `429 Too Many Requests` | Rate limited, with a `Retry-After` header ([ADR 0038](0038-rate-limiting.md)) | `rate_limited` |
| `500 Internal Server Error` | An unexpected failure on our side | `internal_error` |
| `503 Service Unavailable` | Temporarily unavailable, with `Retry-After` when known | `unavailable` |

Choices within that table:

- **400 versus 422.** A request that cannot be read is `400`. A request that can be read but holds invalid values is `422`, with every invalid field listed in `errors`, so a form can show all its problems at once.
- **Not entitled is `403`, not `402`.** `402 Payment Required` is reserved by the HTTP standard with no defined use, and many clients handle it badly. The distinct `not_entitled` code lets the UI offer an upgrade instead of an access-denied message.
- **404 for anything the caller cannot see.** An object in another business, or one the caller has no right to see, gets the same `404 not_found` as an ID that has never existed. Nobody can probe which IDs exist. `403` is used only when the caller can see the object but cannot perform the request.
- **Unknown fields are refused.** A request body containing a field the endpoint does not accept is `400 unknown_field`, never silently ignored. A field the caller may not write, including a field hidden from them ([ADR 0010](0010-api-first.md)), gets exactly the same response, so the error does not reveal that the field exists. Read-only fields, such as `id` or computed totals, count as fields the caller may not write.

### Idempotency

- Clients may send an **`Idempotency-Key`** header on any `POST`, whether it creates an object or performs an action. The key is a value unique to one logical operation, such as a UUID, at most 255 characters.
- The server records the key, scoped to the business and the credential (session or API token), with a hash of the method, path and body, and the response.
  - A repeat with the same key and the same request returns the stored response, with the header `Idempotent-Replayed: true`, and does nothing else.
  - A repeat with the same key but a different request is refused with `422 idempotency_key_reused`.
  - A repeat while the original is still being processed is refused with `409 request_in_progress`. The client retries shortly.
  - Responses with a `5xx` status are not stored, so the request can be retried with the same key.
- Keys are kept for **24 hours**, then hard-deleted as short-lived technical data ([ADR 0018](0018-soft-delete.md)).
- Keys are **optional**. The generated API client ([ADR 0009](0009-frontend-react.md)) sends one on every `POST` automatically, reusing it for retries, so the web app is always protected. The API reference recommends keys to every integrator.
- `GET`, `PATCH` (JSON Merge Patch) and `DELETE` are already safe to repeat, and ignore the header.
- Work started by a request but done later, such as sending an email or syncing to the accounting system, is protected separately by the outbox ([ADR 0023](0023-email-sending.md)).

### JSON types

- **Numbers are numbers.** Amounts, rates, percentages and quantities are plain JSON numbers, never strings, so integrators never need to look up a numeric field's type. Their precision and rounding are decided in [ADR 0036](0036-money-tax-and-time.md).
    - Decimal values have at most 15 significant digits, and integers stay below 2^53, so every value is exact when read as a JavaScript number.
    - The server writes numbers without exponents (`0.0001`, never `1E-4`), and accepts either on input.
    - Clients must not rely on trailing zeros: `1234.50` may arrive as `1234.5`. Amounts are formatted for display by the client, using the currency's minor unit.
    - A value with more decimal places or digits than its field allows is refused with `422`, field code `too_many_decimal_places`, never silently rounded.
- **Percentages are written as percent**: `10` means 10%. Their names end in `Percent`, such as `taxRatePercent` or `discountPercent`.
- **Moments in time** are RFC 3339 strings in UTC, such as `2026-10-11T03:15:00Z`, with names ending in `At`. **Calendar dates** are `YYYY-MM-DD`, with names ending in `Date` ([ADR 0036](0036-money-tax-and-time.md)).
- **Durations** are decimal hours, with names that say so, such as `machineHours: 1.5`.
- **Quantities** sit next to a `unit` field from a fixed list, such as `"quantity": 2.5, "unit": "m2"` ([ADR 0036](0036-money-tax-and-time.md)).
- **Amounts on a document** share the document's `currency` field. They are not wrapped in `{amount, currency}` objects.
- **`null` versus a missing field.** Every field the caller can see is always present in a response, even when it has no value. So:
    - `null` means the field has no value;
    - a missing field means it is not available to this caller: hidden from them ([ADR 0010](0010-api-first.md)), not selected with RQL ([ADR 0016](0016-rql-queries.md)), or behind a feature flag that is off ([ADR 0033](0033-feature-flags.md)).
    - In a `PATCH`, JSON Merge Patch applies: a value sets the field, `null` clears it, and a missing field leaves it unchanged ([ADR 0020](0020-url-structure.md)).
- **No stand-ins for nothing.** `""` and `0` are never used to mean "no value": a discount of `0` is a deliberate value, while `null` means none has been set. An empty string sent for a text field is stored and returned as `null`.
- **Enumerations** are `snake_case` strings, such as `in_production`, never integers. **Booleans** are `true` and `false`, never `"Y"` or `1`. JSON has no `NaN` or `Infinity`, and Signboard never needs them.

### Versioning

- **The major version is in the path**, `/api/v1/…` ([ADR 0020](0020-url-structure.md)). A new major version is created only for breaking changes.
- **Additive changes do not need a new version.** They include new endpoints, new collections, new optional request fields, new response fields, new values in an existing enumeration, new error codes, and new optional headers or query options.
- **Clients must tolerate additions.** They ignore response fields they do not know, and handle enumeration values and error codes they do not know. The API reference states this, and the generated client does it.
- **Breaking changes need a new major version.** They include:
    - removing or renaming an endpoint, field or enumeration value;
    - changing a field's type, format or meaning;
    - making a request field required, or tightening its validation;
    - changing the status code or error code for an existing situation;
    - changing default behaviour, such as default sorting or page size.
- **CI enforces this** by comparing each pull request's OpenAPI contract with `main` and failing on a breaking change unless the pull request is marked as breaking ([ADR 0028](0028-branching-ci-and-releases.md)).
- **Before public launch, `v1` may change freely.** There are no outside integrators yet, and getting the design right matters more. From public launch, `v1` is stable under the rules above.
- **Endpoints and fields behind a feature flag** are not part of the stable contract until their flag is removed ([ADR 0033](0033-feature-flags.md)).
- **Retiring a version.** When a new major version ships, the previous one keeps working for at least **12 months**. Its responses carry the `Deprecation` header ([RFC 9745](https://www.rfc-editor.org/rfc/rfc9745)) and the `Sunset` header ([RFC 8594](https://www.rfc-editor.org/rfc/rfc8594)), and the API changelog announces the dates.
- **Our own UI and API stay compatible across one release.** Because they deploy separately, every API release keeps working with the previous UI build. Changes to the database and API use **expand, then contract**: add the new field or endpoint, move every client to it, then remove the old one in a later release.
- The API version is independent of Signboard's release version ([ADR 0028](0028-branching-ci-and-releases.md)). Webhook payloads are versioned in the events and webhooks ADR.

## Options considered

### Error format

1. **Problem details with our own `type` pages, a stable `code`, a list of field errors and the bare trace ID**: standard, identifies each error, carries machine-readable field errors, and links straight to the logs and audit trail. **Chosen.**
2. **ASP.NET Core's default problem details**, as used by the SoftwareOne Marketplace API: free, standard, and familiar to .NET developers; its `type` only links to the definition of the status code, field errors have no codes, and messages come from the framework.
3. **A custom error envelope**: full control; no standard for clients or tools to recognise.

### Validation status

1. **`422` for invalid values, `400` for unreadable requests**: lets clients tell "fix your client" from "fix your input". **Chosen.**
2. **`400` for both**: simpler, and common; loses that distinction.

### Idempotency keys

1. **Optional on every `POST`, always sent by the generated client**: protects the web app automatically without burdening simple integrations. **Chosen.**
2. **Required on endpoints with outside effects** (sending email, recording payments): stronger for integrations; a hard error for clients that do not send one, and a judgement call per endpoint.
3. **None, relying on clients not to retry**: simplest; duplicates on every network failure.

### Versioning

1. **Major version in the path, additive changes within it, a 12-month overlap for retired versions, free change before launch**: simple to understand and to route, and matches ADR 0020. **Chosen.**
2. **Date-based versions in a header**: fine-grained; much harder to implement, test and document for a small team.
3. **No versioning, change carefully**: the least work; nothing to fall back on when a change must break.

## Consequences

- Each error code needs a page in the API reference, and adding a code means adding its page. The pages also serve as the list of codes for integrators and the front end.
- Validation is written to report every problem at once, with JSON Pointers, rather than stopping at the first.
- Refusing unknown fields means clients must not send back fields they read, such as `id` or computed totals, unless they are writable. The generated client's request types only contain writable fields.
- Idempotency needs a small table and a check in the request pipeline, plus tests for replays, mismatches and concurrent retries.
- Expand-then-contract makes some changes take two releases. It is the price of deploying the UI and API separately.
- JSON serialisation is configured once, so the rules on numbers, dates, `null` and enumerations hold for every endpoint, with contract tests to check them.
- Integrators must be told plainly, in the API reference, to tolerate unknown fields, enumeration values and error codes.
