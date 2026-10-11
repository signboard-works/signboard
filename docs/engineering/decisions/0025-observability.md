# 0025. Observability: logs, traces, metrics and alerts

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Running a multi-tenant service ([ADR 0004](0004-multi-tenant.md)) means finding out what went wrong, for which business, and why, often days after the fact and after a fortnight away from the code. That needs logs, the ability to follow one request through the system, numbers that show when things drift, and alerts when something hurts a business.

Logs are not the audit trail and not product analytics:

| | Audit trail ([ADR 0019](0019-audit-trail.md)) | Observability (this record) | Product analytics ([ADR 0024](0024-product-analytics.md)) |
|---|---|---|---|
| Purpose | A business record of who changed what | Running and fixing the service | How the product is used |
| Read by | Businesses, customers, operations | Operations and developers | Product decisions |
| Business data | Yes, filtered by field visibility | **None** | **None** |
| Kept | For the life of the business | 90 days | Per PostHog plan |

Hosting has not been decided yet, so the decision must not depend on where Signboard runs.

## Decision

### OpenTelemetry throughout

- The back end logs through .NET's standard `ILogger`, and produces **logs, traces and metrics** with the OpenTelemetry SDK, exported with OTLP.
- No code depends on a particular observability product. Where the data goes is configuration only.
- **Destination:** for the hosted service, an AWS Distro for OpenTelemetry collector sends logs to CloudWatch Logs, traces to X-Ray and metrics to CloudWatch, all in Sydney ([ADR 0030](0030-hosting-aws.md)). Development uses a local OpenTelemetry collector and viewer.
- **Self-hosted installations** write structured JSON logs to the console by default, which the container runtime collects, and may point the OTLP exporter wherever they like.

### What is recorded

- **Logs are structured**, never sentences with values pasted in. Every entry carries the trace ID, the tenant, the acting user's ID and the IDs of the objects involved ([ADR 0017](0017-object-ids.md)).
- **The trace ID is the audit trail's correlation ID** ([ADR 0019](0019-audit-trail.md)), so one request can be followed from its log lines and spans to the audit entries it produced.
- **Traces** cover every API request, database query, outbound call (email provider, accounting system) and background job.
- **Metrics** cover request rates, errors and latencies per endpoint, background queue depth, email sending and delivery outcomes, and sign-in success and failure.
- **Levels:** Information for one summary line per request and for significant events; Warning for recoverable problems; Error for failures. Debug is off in production, and can be switched on at runtime without a redeploy.
- **Health endpoints** report whether each service is alive and ready, for the edge router and the container platform. Their paths are set in the deployment design.

### Nothing sensitive

- Logs, traces and metrics contain **no business data and no personal information**: no names, contact details, prices, free text, request or response bodies, email contents, tokens or passwords. Only IDs, kinds, states, counts and durations.
- This is enforced by a redaction step in the telemetry pipeline, not left to care, and tested ([ADR 0012](0012-testing.md)).
- Because telemetry holds only IDs, operations staff can search it across every business without breaching the rule that they cannot see a business's data unless they join its account ([ADR 0010](0010-api-first.md)).

### Front-end errors

- Errors in the browser app are sent to **PostHog's error tracking**, alongside product analytics ([ADR 0024](0024-product-analytics.md)), under the same rules: user identified by object ID, no business data.
- Error messages and stack traces are scrubbed of anything that could hold business data before they are sent.

### Retention

Logs, traces and metrics are kept for **90 days**, to be reviewed once real volumes and costs are known.

### Alerts

- Alerts go **by email**.
- **Nothing pages anyone out of hours.** Every alert waits until the morning. This is a deliberate choice for a weekends-only project, and the status of the service is shown honestly to businesses when something is wrong ([ADR 0013](0013-work-in-the-open.md)).
- Initial alerts:
  - error rate above normal for any endpoint;
  - email sending failures, or a rise in bounces or complaints, because sign-in depends on email ([ADR 0023](0023-email-sending.md));
  - a spike in failed sign-ins;
  - the API or identity service failing its health checks;
  - background queues growing faster than they drain.

## Options considered

1. **OpenTelemetry, all four signals, destination chosen with hosting**: one standard instrumentation, any backend, nothing to rework when hosting is chosen. **Chosen.**
2. **Serilog with a product-specific sink**: popular and pleasant to use; ties the code more closely to one destination, and traces and metrics need separate tooling.
3. **Logs only**: least effort; following a slow or failing request across the API, database and background jobs is guesswork.
4. **Sentry for front-end errors**: excellent at it, with a free tier; another vendor, when PostHog already covers it.
5. **Out-of-hours paging**: problems fixed sooner; not sustainable for one part-time maintainer.

## Consequences

- An incident overnight is not looked at until the morning. Businesses should know this, and the status page and incident reviews ([ADR 0013](0013-work-in-the-open.md)) must be honest about it.
- The backend can change later with a collector configuration change, because the code only speaks OTLP.
- Every log statement written uses structured fields with IDs only. Reviews check this, and the redaction step backs it up.
- Ninety days of logs and traces may cost more than a free tier allows once there are several businesses. Retention is reviewed then.
