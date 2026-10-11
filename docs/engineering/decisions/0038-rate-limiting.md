# 0038. Rate limiting

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |


## Context

Every business shares one hosted service ([ADR 0004](0004-multi-tenant.md)), and ADR 0004 already says per-tenant rate limiting is needed so that one business cannot slow down the rest. The same public API serves the web app, customers and integrations ([ADR 0010](0010-api-first.md)), so a buggy integration in a tight loop, or a script exporting every quote one at a time, would otherwise reach the database at full speed.

Some requests are attractive to attackers whatever their volume: guessing an emailed sign-in code, trying leaked passwords against many accounts, creating accounts in bulk, or using Signboard to send email to people who did not ask for it ([ADR 0023](0023-email-sending.md)).

Signboard is also a client of other services, but pacing our own outgoing calls is already decided in [ADR 0035](0035-retries-and-resilience.md). This ADR is about requests coming in.

The limits below are starting values, to be tuned from real traffic.

ADR 0029 already defines the response: `429 Too Many Requests`, code `rate_limited`, with `Retry-After`. The CloudFront flat-rate plan ([ADR 0030](0030-hosting-aws.md)) includes AWS WAF with IP-based rate limiting on every tier.

## Decision

### Three layers

| Layer | Where | Protects against | Counts |
|---|---|---|---|
| **1. Edge** | AWS WAF on CloudFront ([ADR 0030](0030-hosting-aws.md)) | Floods and denial-of-service attempts, before they reach our servers | Requests per IP address over 5 minutes |
| **2. Fair use** | ASP.NET Core's rate limiting middleware, in the API and identity service | One user, integration or business using more than its share | Requests per credential and per business |
| **3. Abuse-sensitive actions** | The application, counted in PostgreSQL | Guessing codes and passwords, bulk sign-ups, sending spam | Attempts per email address, account and IP address |

### Layer 1: the edge

- A WAF rate-based rule blocks an IP address that sends more than **10,000 requests in 5 minutes** to the whole site, and more than **300 in 5 minutes** to the identity service.
- The limits are generous on purpose: all the staff at a shop usually share one IP address, and the web app makes many small requests. The edge only stops floods. Fairness is layer 2's job.
- AWS's managed rules for common attacks, included in the plan, are on, within the plan's limit on the number of rules (25 on Pro).
- Blocked requests get a `429` in the same problem details shape as the API's, using WAF's custom responses, which the Pro plan includes. Non-production environments, on the Free plan, get WAF's default response.
- Self-hosted installations have no WAF; they can put their own proxy in front, and layers 2 and 3 still apply.

### Layer 2: fair use

- Built with **ASP.NET Core's rate limiting middleware** (part of ASP.NET Core, MIT), using token buckets, so short bursts, such as a page loading several grids, are allowed while sustained load is not.
- Policies are named and defined in one place, like the resilience pipelines ([ADR 0035](0035-retries-and-resilience.md)). Endpoints never set their own limits in code.
- Requests are limited after authentication, so each is counted against **who is calling**, and also against the **business** it acts in ([ADR 0010](0010-api-first.md)).

| Limit | Sustained | Burst |
|---|---|---|
| Per user session (one membership) | 20 requests a second | 200 |
| Per API token | 10 requests a second | 100 |
| Per business, all users and tokens together | 100 requests a second | 1,000 |
| Concurrent requests per credential | 20 at once | |
| Live update streams ([ADR 0032](0032-events-webhooks-live-updates.md)) | 10 open per user, 500 per business | |
| Requests before sign-in, per IP address | 5 requests a second | 50 |

- **Expensive work is limited by the job queue, not by request counts.** Exports, imports, PDF generation and bulk changes run as jobs with per-type concurrency limits and per-business fairness ([ADR 0034](0034-background-work.md)), so asking for ten exports queues them rather than running them all at once. Collection queries are already capped at 1,000 items a page ([ADR 0016](0016-rql-queries.md)).
- **Counters are kept in memory in each API task**, not in a shared store. Each task enforces its share of each limit, based on the number of tasks running, so the overall limit is approximate. A fair-use limit does not need to be exact, and this avoids adding Redis, which Signboard otherwise does not need ([ADR 0034](0034-background-work.md)).
- **Health checks are exempt.** Nothing else is, including operations users and Signboard's own web app.

### Layer 3: abuse-sensitive actions

These limits must be exact and survive restarts and deploys, so they are counted in PostgreSQL, in the identity service and the `system` module.

| Action | Limit |
|---|---|
| Emailed sign-in codes and links ([ADR 0021](0021-authentication.md)) | 5 sent to one address an hour, and 20 from one IP address an hour |
| Guessing an emailed code | 5 wrong guesses, then the code is spent and a new one must be requested |
| Password and second-factor attempts | After 5 failures on an account, each further attempt waits longer, up to 15 minutes. Accounts are never locked out, because a lockout would let anyone block someone else's account |
| Sign-in attempts across many accounts from one IP address | 50 failures an hour, which catches attempts to try leaked passwords |
| Sign-ups | 10 from one IP address an hour |
| Email sent by a business (hosted service only) | A daily limit per business, 1,000 at first, raised by an operations user on request. It protects the sending reputation every business shares ([ADR 0023](0023-email-sending.md)) |
| Invitations to join an account | 50 a day per account |

- Sign-in responses do not reveal whether an address has an account, whether or not a limit has been reached.
- If sign-ups or sign-ins are abused despite these limits, the WAF's CAPTCHA challenge, included in the Pro plan, can be added to those pages without code changes.

### Responses

- Every refusal is `429 Too Many Requests`, code `rate_limited`, with a `Retry-After` header in seconds ([ADR 0029](0029-api-errors-idempotency-versioning.md)). The `detail` says which limit was reached, such as "Too many requests for this API token."
- Signboard's own clients slow down and retry automatically, honouring `Retry-After` ([ADR 0035](0035-retries-and-resilience.md)). The web app shows a message only if retries do not succeed.
- **The `RateLimit` and `RateLimit-Policy` headers**, which tell a client how much of its limit is left before it is refused, are being standardised by the IETF but are still a draft. We will add them when they are published as an RFC. Adding headers is not a breaking change ([ADR 0029](0029-api-errors-idempotency-versioning.md#versioning)).

### Changing limits

- Limits are configuration per environment, not code, and are documented in the API reference so integrators know them.
- An operations user can raise a business's limits for a period, such as during a data migration ([migration PRD](../requirements/prd-migration.md)). The change is audited ([ADR 0019](0019-audit-trail.md)).
- On the hosted service, different editions may get different fair-use limits. That is decided in the entitlements ADR.
- Self-hosted installations ship with the same defaults, and can change or turn off layer 2 in configuration. Layer 3 cannot be turned off, only adjusted.

### Visibility

- Every refusal is counted in metrics by layer, limit and business, and logged with IDs only ([ADR 0025](0025-observability.md)).
- An alert is raised when a business keeps reaching its business-wide limit, which usually means a runaway integration, and when layer 3 refusals rise sharply, which may mean an attack.

## Options considered

### Where to limit

1. **The edge for floods, the application for fairness and abuse**: each layer does what it is good at; the edge cannot see who is calling, and the application cannot absorb a flood. **Chosen.**
2. **The edge only**: nothing to build; limits by IP address alone are unfair to shops whose staff share one, and cannot limit per business, token or email address.
3. **The application only**: one place; floods still reach our servers and use their capacity.
4. **An API gateway with usage plans**, such as Amazon API Gateway: limits per key built in; another hop and service in front of the API, a different answer for self-hosters, and still no per-business limit.

### Where to keep fair-use counters

1. **In memory in each task, sharing out each limit**: no extra service, no database writes per request; limits are approximate. **Chosen.**
2. **Redis (Amazon ElastiCache)**: exact limits across tasks; another service to run, pay for and secure, and for self-hosters to install, used only for this.
3. **PostgreSQL for every request**: exact, and no new service; a database write on every request, which is a poor trade for a fair-use limit.

## Consequences

- Each API task needs to know how many tasks are running, which the deployment design provides.
- The client IP address comes from CloudFront's viewer address header, trusted only because the API is reachable only through CloudFront ([ADR 0030](0030-hosting-aws.md)). Self-hosted installations configure which proxy to trust.
- The identity service gains attempt counters for codes, passwords and sign-ups, designed with the authentication design.
- Bulk operations that integrators and the migration need, such as importing many customers, must exist as bulk or import endpoints that run as jobs, so nobody has to make thousands of single requests.
- Every limit has an integration test that reaches it and checks the `429` and `Retry-After`, and the sign-in limits have tests for each attack they are meant to stop ([ADR 0012](0012-testing.md)).

