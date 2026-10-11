# 0035. Retries and resilience

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

## Context

Signboard depends on services it does not control: accounting systems ([ADR 0005](0005-accounting-as-plugin.md)), email ([ADR 0023](0023-email-sending.md)), integrators' webhook endpoints ([ADR 0032](0032-events-webhooks-live-updates.md)), and AWS itself ([ADR 0030](0030-hosting-aws.md)). Each will sometimes be slow, refuse a request, or be down. The database can also be briefly unavailable, such as during a Multi-AZ failover.

Retrying is the obvious answer, and done carelessly it causes harm: a request retried that was not safe to repeat creates a second invoice; retries stacked inside retries multiply into a storm that hammers a service just as it recovers; and a call with no timeout ties up a worker indefinitely. Background jobs already retry over long periods ([ADR 0034](0034-background-work.md)), so the rules for retrying individual calls must fit with that.

## Decision

### API requests do not wait on outside services

Anything that calls an outside service, such as sending email, syncing to accounting, delivering webhooks or rendering PDFs, runs as a background job ([ADR 0034](0034-background-work.md)), never inside an API request. The few calls an API request still makes are fast and inside AWS, such as presigning an S3 URL or reading a secret. This keeps the API's response times predictable and puts nearly all retrying in the worker.

### Two layers, never stacked

| Layer | Where | Handles | Limits |
|---|---|---|---|
| **1. Short retries** | Around one outside call, inside a job | Blips: a dropped connection, a single timeout, a brief `503` | About 3 attempts, a few seconds in total |
| **2. Job retries** | The job queue ([ADR 0034](0034-background-work.md)) | Outages: a service down for minutes or hours | Backoff over minutes to days, then the job's failed state |

No call is retried at more than one level within a layer. A library that retries for itself, such as the AWS SDK, is not wrapped in further retries.

### What is retried

- **Every outside call has a timeout**, per attempt and in total.
- **Only temporary failures are retried:** network errors, timeouts, and the HTTP statuses `408`, `429`, `502`, `503` and `504`. Other `4xx` responses are never retried, because the request itself is wrong. A `500` is retried only for operations that are safe to repeat.
- **On `429` or `503` with a `Retry-After` header**, the next attempt waits at least that long.
- **Only operations that are safe to repeat are retried:** reads, and writes the destination can de-duplicate, such as requests carrying an idempotency key. Any other write is retried only where its design doc shows that a repeat is harmless.
- **Backoff is exponential with jitter**, so retries from many jobs spread out rather than arriving together.

### Protecting outside services, and ourselves

- **A circuit breaker per destination**, such as each business's accounting connection. When a destination keeps failing, calls to it stop for a short period. Jobs that meet an open circuit reschedule themselves instead of using up their attempts.
- **A client-side rate limiter per destination**, set to that service's published limits, so we pace our own calls rather than being refused.

### Policies are defined centrally

Resilience policies are **named pipelines per destination**, such as `accounting`, `email` and `webhooks`, defined in one place and applied through dependency injection. Feature code never writes its own retry loop.

### Tools

| Calls to | Resilience from |
|---|---|
| Outside services over HTTP | **Microsoft.Extensions.Http.Resilience** (MIT): the standard resilience handler for `HttpClient`, built on Polly, with timeouts, retries, circuit breaker and rate limiting |
| Anything else that needs it | **Polly** 8 (BSD-3-Clause) directly |
| AWS services | The AWS SDK's built-in retry mode (`standard`), not wrapped |
| PostgreSQL | EF Core's retrying execution strategy for Npgsql, which retries the whole transaction on temporary errors, such as during a Multi-AZ failover ([ADR 0030](0030-hosting-aws.md)) |

Because the execution strategy retries whole transactions, code that opens its own transaction runs it inside the strategy, so a retry repeats the complete unit of work, never half of it.

### Our own clients retry too

The web app and the generated API client retry automatically, with backoff, on network errors, `429` and `503`, for reads and for writes that carry an idempotency key ([ADR 0029](0029-api-errors-idempotency-versioning.md)). Concurrency checks ([ADR 0031](0031-concurrency.md)) keep a repeated write safe even when the first one had in fact succeeded. The API reference gives integrators the same advice.

### Visibility

Every retry, timeout and change in a circuit breaker's state is recorded in metrics and logs ([ADR 0025](0025-observability.md)). An open circuit raises an alert.

### Testing

Resilience is tested, not assumed ([ADR 0012](0012-testing.md)). Integration tests use Polly's built-in chaos strategies to inject faults, delays and outages, and **WireMock.Net** (Apache 2.0) to stand in for outside APIs, so each named pipeline is shown to retry, time out, back off, open its circuit and honour `Retry-After` as designed.

## Options considered

1. **Microsoft.Extensions.Http.Resilience and Polly, with central named pipelines and two layers**: the standard for .NET, maintained by Microsoft and the Polly project, with fault injection for testing built in. **Chosen.**
2. **Polly alone, configured by hand everywhere**: the same engine; more code, and policies drift between features.
3. **Hand-written retry loops**: no dependency; inconsistent, rarely tested, and usually missing jitter, timeouts and circuit breaking.
4. **No in-process retries, job retries only**: simpler; every momentary blip costs a full job retry and its delay.

## Consequences

- Each outside integration's design doc names its pipeline, its timeouts and limits, and which of its operations are safe to repeat.
- Writing to an outside service that cannot de-duplicate needs care in its design doc: either a check before retrying, or accepting the job's failed state for a person to look at.
- Code that opens a database transaction runs it through the execution strategy, which code review and an architecture test check.
- Every pipeline has fault-injection tests. Adding a destination means adding its pipeline and its tests.
