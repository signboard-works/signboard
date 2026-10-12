# 0033. Feature flags

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

## Context

Signboard uses trunk-based development: unfinished work merges to `main` and must not be visible until it is ready ([ADR 0028](0028-branching-ci-and-releases.md)). New features should also reach one business first, such as the design partner shop, before everyone. And during an incident, it must be possible to switch off one part of the system, such as accounting sync, without a deploy.

Feature flags are easy to add and hard to remove. Left alone, they pile up, every old flag doubles the paths through the code, and nobody remembers which ones still matter. Temporary often becomes permanent.

Feature flags are not:

- **Entitlements**: what a business's plan includes. That is permanent and commercial, and decided in [ADR 0046](0046-entitlements.md).
- **Business settings**: how a business wants Signboard to behave ([ADR 0006](0006-configuration-over-customisation.md)).
- **Permissions**: what a user may do ([ADR 0022](0022-authorisation-and-roles.md)).
- **Experiments** (A/B tests), which are out of scope.

## Decision

### Kinds of flag

| Kind | Purpose | Lifetime |
|---|---|---|
| **Release flag** | Hides unfinished or not-yet-launched work, and previews it with chosen businesses first | Temporary: deleted soon after full rollout |
| **Kill switch** | Switches off part of the system during an incident, such as accounting sync, webhook delivery or one integration | Long-lived. Named `ops.*` |

### Declared in code

- Every flag is **declared in code**, like permissions ([ADR 0022](0022-authorisation-and-roles.md)): a name such as `sales.quote-templates` (module, then feature), a description, an owner, the date it was created, and for release flags a **remove-by date**, 90 days after creation by default.
- A flag not declared in code does not exist, whatever is stored.
- **Off is the safe state.** A release flag with no stored state, or whose state cannot be read, is off. A kill switch leaves its feature running unless it is deliberately thrown.

### State and evaluation

- Flag state is stored in PostgreSQL, per environment, in the `system` module. A flag can be on for everyone, for named businesses, for named users, or for a percentage of businesses.
- A percentage rollout hashes the business's ID, so each business is consistently in or out as the percentage rises.
- **Customer users get their business's flags**, so the customer portal matches what the business sees.
- We use **Microsoft.FeatureManagement** (MIT) for evaluation, with a provider that reads state from PostgreSQL. Code never calls the library directly: it calls Signboard's own `IFeatures` interface, so the library could be replaced without touching features.
- Flag state is cached in memory in every API task and the worker. A change to a flag is audited ([ADR 0019](0019-audit-trail.md)) and raises an event ([ADR 0032](0032-events-webhooks-live-updates.md)), so caches refresh, and open browsers update within seconds without a reload.

### Who switches flags

- **Operations users** switch flags, through the operations app, with a permission of their own ([ADR 0022](0022-authorisation-and-roles.md)).
- Business administrators cannot. A preview programme, where a business chooses to try upcoming features early, may come later as its own feature.

### When a flag is off, the feature does not exist

- The API behaves as if the feature had never been built: its endpoints return `404 not_found` ([ADR 0029](0029-api-errors-idempotency-versioning.md)), its fields are omitted from responses, and it is left out of RQL allow-lists ([ADR 0016](0016-rql-queries.md)).
- **Flagged endpoints and fields are not part of the stable API contract until released.** They are marked in the OpenAPI document and left out of the public API reference until their flag is removed, so the rule that `v1` only changes by adding things holds ([ADR 0029](0029-api-errors-idempotency-versioning.md)).
- The web app asks `GET /api/v1/system/features`, which returns the flags that are on for the current membership, and hides whatever is off. The check that matters is always the API's.

### Retiring flags

- **CI fails when a release flag is past its remove-by date.** The date can be extended, but only in a pull request that says why.
- Retiring a flag deletes the flag and the old code path in the same pull request.
- Kill switches have no remove-by date. They are listed in the API reference so their existence is visible.

### Environments

| Environment | Release flags |
|---|---|
| Preview (one per pull request) | All on, so the work can be tried ([ADR 0028](0028-branching-ci-and-releases.md)) |
| Staging | Set per flag by the maintainer |
| Production | Off until switched on |
| Self-hosted | As shipped in each release, and overridable in configuration. There is no operations app |

### Testing

Code behind a flag is tested with the flag both on and off ([ADR 0012](0012-testing.md)). End-to-end tests run against the flag states of the environment they target.

## Options considered

### Evaluation

1. **Microsoft.FeatureManagement behind our own interface, state in PostgreSQL**: mature, MIT-licensed, native to .NET, handles targeting and percentages; state stays inside Signboard and works offline and self-hosted. **Chosen.**
2. **OpenFeature**: a vendor-neutral standard API with .NET and React SDKs; at our size an extra layer with no provider we would use, since we would write our own anyway.
3. **PostHog feature flags**: already in use for analytics, with flags in its free tier; the API would depend on an outside service in the EU to decide behaviour, and self-hosted installations send nothing to PostHog ([ADR 0024](0024-product-analytics.md)).
4. **Our own evaluation code**: no dependency; reinvents targeting and percentage rollout.

### Retiring flags

1. **A remove-by date on every release flag, enforced by CI**: flags cannot quietly become permanent; extensions are deliberate and visible. **Chosen.**
2. **A periodic review of old flags**: lighter; depends on someone remembering, which is how flags pile up.

## Consequences

- Every new feature's design doc says whether it ships behind a flag, and names the flag.
- The flag registry in code doubles as the list of what is in progress. The operations app shows each flag's state per environment and its remove-by date.
- Unreleased endpoints can be built and tested on `main` without becoming part of the public contract.
- Removing a flag is real work, and CI makes it impossible to put off indefinitely.
- Each flag adds a path to test. Keeping flags short-lived keeps that cost small.
