# 0046. Entitlements

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

> **Draft for review.** Written from Claude's recommendations. The choices still open are listed under [Questions for review](#questions-for-review).

## Context

The hosted service will offer editions: plans at different prices that include different features and limits. Which editions exist and what they cost is decided in [ADR 0047](0047-sign-up-plans-and-billing.md). This ADR decides the mechanism: how Signboard knows what a business's edition includes, and what happens when a request goes beyond it.

Three things are already decided, by Stuart on 2026-10-11:

- **Editions apply only to the hosted service.**
- **Self-hosted installations get every feature**, with no limits.
- **The entire product is open source.** There is no open core: no feature is held back from the code, and nothing is unlocked by a licence key.

An entitlement is easily confused with three other things, each decided elsewhere:

| | Question it answers | Lifetime | Decided in |
|---|---|---|---|
| **Entitlement** | What does this business's plan include? | Permanent, commercial | This ADR |
| **Feature flag** | Is this feature released yet? | Temporary, or an `ops.*` kill switch | [ADR 0033](0033-feature-flags.md) |
| **Setting** | How does this business want it to behave? | Permanent, chosen by the business | [ADR 0006](0006-configuration-over-customisation.md) |
| **Permission** | May this user do this? | Permanent, by role | [ADR 0022](0022-authorisation-and-roles.md) |

[ADR 0029](0029-api-errors-idempotency-versioning.md) already reserves `403 not_entitled` for a feature the business's plan does not include, so the UI can offer an upgrade instead of an access-denied message.

## Decision

### Two kinds of entitlement

| Kind | Example | Value |
|---|---|---|
| **Feature** | `sales.quote-templates`, `system.webhooks`, `billing.accounting-sync` | Included or not |
| **Limit** | `accounts.members`, `system.file-storage` ([ADR 0027](0027-file-storage.md)), `notifications.emails-per-day` ([ADR 0038](0038-rate-limiting.md)) | A maximum, with a unit |

- **Declared in code**, like permissions and feature flags: a name (module, then feature), a description, its kind, and for a limit its unit. An entitlement not declared in code does not exist.
- **Editions are configuration**, stored in PostgreSQL in the `system` module: each edition is a set of included features and limit values. Editions are created and changed through the operations app, and their names and prices belong to [ADR 0047](0047-sign-up-plans-and-billing.md).
- Fair-use rate limits ([ADR 0038](0038-rate-limiting.md)) may also differ by edition. They are configured with the edition, but enforced by the rate limiter, not by this mechanism.

### What is never gated

Some things are in every edition, because Signboard's principles depend on them:

- **Security**: passkeys, second factors, sessions and the audit trail ([ADR 0021](0021-authentication.md), [ADR 0019](0019-audit-trail.md)).
- **Data export**: a business can always take all of its data out ([vision](../../vision.md), [ADR 0044](0044-privacy-and-data-export.md)).
- **Accessibility** ([ADR 0042](0042-accessibility.md)).
- **The API.** Entitlements gate features, never channels. A feature included in an edition can be used through the web app and the API alike ([ADR 0010](0010-api-first.md)).

### Evaluation

- Each business has one **edition assignment** at a time, with an optional end date and the edition that follows it, which is how trials work.
- A business's **effective entitlements** are its edition's, plus any overrides. They are worked out on the server and cached per business. A change raises an event ([ADR 0032](0032-events-webhooks-live-updates.md)), so caches and open browsers refresh.
- Every endpoint and field group that belongs to an entitled feature **declares the feature it needs**, next to its permission ([ADR 0022](0022-authorisation-and-roles.md)). The check runs centrally on every request, in this order:
    1. **Feature flag** ([ADR 0033](0033-feature-flags.md)): a feature that is not released does not exist, so the request gets `404`.
    2. **Permission**: a user without the permission gets `403 forbidden`, whatever the plan.
    3. **Entitlement**: a business whose plan does not include the feature gets `403 not_entitled`.
- **Customer users get their business's entitlements**, as they get its flags ([ADR 0033](0033-feature-flags.md)).
- **Self-hosted installations** run a built-in **everything** edition: every feature, no limits. It is defined in code, needs no licence key and contacts no outside service.

### In the API

- An entitled feature **exists and is documented** whether or not a business's plan includes it, unlike an unreleased feature behind a flag. The API reference shows which editions include each endpoint, from a marker in the OpenAPI contract.
- **Reading:** fields that belong to a feature the plan does not include are omitted from responses, like fields the caller cannot see ([ADR 0010](0010-api-first.md)). This keeps every response the same shape for every caller with the same visibility.
- **Writing:** a request that uses an endpoint or writes a field of a feature the plan does not include gets `403 not_entitled`, naming the feature, rather than `400 unknown_field`. The feature's existence is public, so there is nothing to hide, and the UI can offer the upgrade.
- **Limits:** a request that would go beyond a limit gets `403 limit_reached`, a new error code ([ADR 0029](0029-api-errors-idempotency-versioning.md)), with the limit's name, its maximum and the current usage in the problem details.
- `GET /api/v1/system/entitlements` returns the current business's edition, its features, and its limits with current usage, for the UI. Customer users get only which features are included, without edition names or usage.

### Limits

- Limits are checked when something is created, in the same transaction, under a lock per business and limit, so two requests at once cannot both take the last place. File storage is checked when an upload starts, so one upload can take a business slightly over.
- At **80% of a limit**, business administrators see a notice in the app and get one email.
- **Usage is counted from the data itself**, such as active memberships or the size of stored files ([ADR 0027](0027-file-storage.md)), not from a separate counter that could drift. The email limit uses the counts [ADR 0038](0038-rate-limiting.md) already keeps.

### Downgrades and expiry

- **Nothing is ever deleted or hidden because of a plan change.**
- A feature the new edition does not include becomes **read-only**: its data can still be seen and exported, but not changed or added to.
- Over a limit, everything that exists keeps working, but nothing more can be added until usage is back under the limit. For example, all existing members can still sign in, but no one new can be invited.

### In the UI

- **Business administrators** see features outside their plan, marked as such, with an offer to upgrade.
- **Other staff** see the same mark, with a note to ask an administrator.
- **Customer users** never see a feature their business's plan does not include, and never see an upgrade offer.

### Overrides

- An **operations user** can give one business an override: a feature included, or a limit raised, with an end date and a reason. It is audited ([ADR 0019](0019-audit-trail.md)), and the business's administrators can see it.

## Options considered

1. **Our own entitlements in PostgreSQL, declared in code and evaluated with permissions**: fits tenancy, authorisation and the self-hosted everything edition exactly; nothing extra to run or pay for. **Chosen.**
2. **The billing provider's entitlements**, such as Stripe's Entitlements API: maintained for us; ties every request to the billing provider's data, and self-hosted installations have no billing provider. The billing provider can still tell us which edition is paid for ([ADR 0047](0047-sign-up-plans-and-billing.md)).
3. **Feature flags for entitlements**: one mechanism; ruled out by [ADR 0033](0033-feature-flags.md), because flags are temporary and entitlements are permanent and commercial.
4. **A third-party entitlement service**: a ready-made dashboard; another paid service and another dependency, for a handful of editions.
5. **Licence keys**: the usual way to gate self-hosted features; ruled out by the decision that the whole product is open and self-hosted gets everything.

## Consequences

- Every feature's design doc names its entitlement, if it has one, and every endpoint and field group declares it. An automated test checks every declared entitlement is used, and every use is declared.
- Every entitled endpoint has tests with the feature included and not included, and every limit has tests for reaching it, including two requests at once ([ADR 0012](0012-testing.md)).
- [ADR 0029](0029-api-errors-idempotency-versioning.md) gains the `limit_reached` code, and the API reference shows the editions for each endpoint.
- The operations app needs screens for editions, assignments and overrides. The web app needs the locked-feature mark and the upgrade offer.
- Self-hosted installations need no configuration for entitlements at all.

## Questions for review

1. **Writing to a feature outside the plan gets `403 not_entitled`**, rather than the `400 unknown_field` used for fields hidden from the caller. Agreed, given that entitled features are public?
2. **Locked features are shown to staff**, with an upgrade offer for administrators, and hidden from customers. Or hide them from everyone?
3. **A limit gives `403 limit_reached`**. Or `409 Conflict` or `422`?
4. **Downgrades never delete or hide anything**: removed features become read-only, and over a limit nothing more can be added. Or lock data after a grace period?
5. **Never gated**: security, data export, accessibility and the API. Anything to add or remove?
6. **A warning at 80%** of each limit, in the app and by one email. Or no warning?
7. **Operations overrides** with an end date and a reason, visible to the business. Or no overrides, only edition changes?
