# 0034. Background work

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

## Context

A lot of Signboard's work happens outside an API request, in the background worker ([ADR 0026](0026-modular-monolith.md)):

| Kind | Examples | Comes from |
|---|---|---|
| **Event handlers** | A module reacting to `sales.quote.accepted`; live-update fan-out; webhook deliveries | The event outbox ([ADR 0032](0032-events-webhooks-live-updates.md)) |
| **One-off jobs** | Sending an email ([ADR 0023](0023-email-sending.md)), generating a quote PDF, syncing an invoice to accounting ([ADR 0005](0005-accounting-as-plugin.md)), recording a malware scan result ([ADR 0027](0027-file-storage.md)), imports and exports | Enqueued by an API request or another job |
| **Delayed jobs and retries** | Webhook retries over about three days, email retries | Scheduled for a later time |
| **Recurring jobs** | Deleting expired events, idempotency keys and logs; creating the next audit partitions ([ADR 0019](0019-audit-trail.md)); overdue-invoice reminders; polling accounting for payments | A schedule |

This work must not be lost when something fails, must not run twice in harmful ways, must respect tenancy exactly as the API does ([ADR 0004](0004-multi-tenant.md)), and must not let one business's heavy work delay everyone else's. Self-hosters run the same worker with Docker Compose, so it must not need services beyond PostgreSQL.

## Decision

### A job queue in PostgreSQL

- Background work runs from a **job queue in PostgreSQL**, built in the `system` module. There is no Redis and no message broker.
- Workers claim jobs with `SELECT … FOR UPDATE SKIP LOCKED`, so several worker tasks share one queue without two of them taking the same job.
- A new job wakes workers immediately with `LISTEN/NOTIFY`. Workers also poll, so nothing waits long if a notification is lost.
- The queue shares its machinery with the event outbox's dispatcher ([ADR 0032](0032-events-webhooks-live-updates.md)): delivering events to their consumers is itself work on this queue.
- The queue is our own code: a few hundred lines of a well-known pattern, tested against real PostgreSQL with Testcontainers ([ADR 0012](0012-testing.md)). Cron expressions are parsed with the **Cronos** library (MIT).

### Enqueued with the change

A job is enqueued **in the same database transaction** as the change that needs it. If the change rolls back, the job never existed; if it commits, the job is guaranteed to exist. A quote cannot be marked as sent without its email job, or the other way round.

### Every job carries its context

- Each job records the business it belongs to, the acting user or credential, and the trace ID ([ADR 0025](0025-observability.md)).
- Before running a job, the worker sets the tenant context, so row-level security and EF Core query filters apply exactly as they do in the API ([ADR 0004](0004-multi-tenant.md)). Logs and traces carry the same IDs, so a job can be followed back to the request that caused it.
- System jobs that work across businesses, such as retention clean-up, are declared as such explicitly, and are the only jobs that run without a tenant.

### Running jobs

- **Delivery is at least once.** Every job handler must be safe to run twice, using object revisions ([ADR 0031](0031-concurrency.md)), idempotency keys or its own checks.
- **Retries** use exponential backoff with jitter, with a maximum number of attempts set per job type. Retries of individual calls to outside services follow [ADR 0035](0035-retries-and-resilience.md).
- **Failed jobs** move to a failed state after their last attempt. They are never silently dropped. They are listed in the operations app, through the API, with actions to retry or discard, and they raise an alert ([ADR 0025](0025-observability.md)).
- **Fairness and ordering:**
    - **Per business:** jobs are claimed fairly across businesses, so one business's large import cannot starve another's emails.
    - **Per key:** jobs that must not overlap share a key and run one at a time, such as one accounting sync per business.
    - **Per job type:** each type has a concurrency limit, so heavy jobs such as PDF generation and imports cannot swamp the worker.
- **Long jobs** (imports, exports, large PDFs) work in chunks, record their progress, and can be cancelled.
- **Shutting down cleanly.** When ECS stops a worker task, such as during a deploy ([ADR 0030](0030-hosting-aws.md)), the worker stops claiming new jobs, finishes what it can within the stop timeout, and releases the rest. Jobs interrupted any other way are reclaimed after a timeout and retried.

### Recurring jobs

- Recurring jobs are declared in code with a cron expression, and run **once per scheduled time** however many worker tasks are running.
- Schedules a business would notice, such as "send reminders at 9 am", run in **the business's time zone**. How time zones are stored is decided in the money, tax and time ADR.

### Notifications from AWS

SES delivery events ([ADR 0023](0023-email-sending.md)) and GuardDuty scan results ([ADR 0027](0027-file-storage.md)) arrive on an **Amazon SQS queue**, through SNS and EventBridge. The worker reads that queue and turns each message into an internal job. This adapter is the only AWS-specific part of background work. Self-hosted installations feed the same job types from SMTP and ClamAV instead.

### Housekeeping and scale

- Completed jobs are kept for **7 days** for troubleshooting, then hard-deleted as short-lived technical data ([ADR 0018](0018-soft-delete.md)). Failed jobs are kept until someone retries or discards them.
- **Monitoring** happens in the operations app, through the API: queue depth and age per job type, failures, and running jobs. There is no separate library dashboard, which would be a second admin interface with its own security outside the API ([ADR 0010](0010-api-first.md)).
- **One worker service** to start, scaled by adding tasks. If heavy job types slow everything else, they move to a worker pool of their own, which is a configuration change.

## Options considered

1. **Our own queue on PostgreSQL**: fits same-transaction enqueueing, tenant context and row-level security, and per-business fairness and per-key ordering exactly; shares code with the event outbox; nothing extra to run. We own its bugs, which tests against real PostgreSQL contain. **Chosen.**
2. **TickerQ** (MIT or Apache 2.0, active): native to EF Core, so jobs can share our transaction, and has a dashboard; a younger library, and fairness and per-key ordering would still be our own code on top.
3. **Hangfire** (LGPL v3, with paid Pro features): mature and widely used; PostgreSQL support is a separate community package, the licence is copyleft unlike Signboard's Apache 2.0 ([ADR 0003](0003-licence.md)), and some useful features are paid.
4. **Wolverine** (MIT, active): excellent at outbox, handlers and scheduling; a whole framework with its own conventions, which would reshape how modules are written.
5. **Quartz.NET** (Apache 2.0): strong at schedules; weak as a work queue.
6. **A message broker** (Amazon SQS, RabbitMQ): standard at large scale; another service to run and for self-hosters to install, and enqueueing could not share the database transaction without an outbox anyway.

## Consequences

- The `system` module owns the queue, the outbox dispatcher, recurring schedules and the SQS adapter. Other modules enqueue jobs and declare handlers through its interface.
- Every job type declares its handler, retry policy, concurrency limit and, where needed, its ordering key. The design doc for each feature lists the job types it adds.
- Every job handler has tests for running twice, for failure and retry, and for running under the right tenant, including a cross-tenant case ([ADR 0012](0012-testing.md)).
- The operations app needs a jobs view. Its API is designed with the operations app.
- The deployment design sets the worker's stop timeout, its task count, and the SQS queue with a dead-letter queue of its own.
