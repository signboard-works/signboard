# Architecture Decision Records

Each significant decision gets one short, numbered record: the context, the decision, the options considered, and the consequences. Records are never deleted. During initial design, before any code implements a decision, its record may be edited directly. After that, a changed decision gets a new record that supersedes the old one.

| # | Decision | Status |
|---|---|---|
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions | Accepted |
| [0002](0002-docs-as-code.md) | Documentation as code, in the repository | Accepted |
| [0003](0003-licence.md) | Open-source licence: Apache 2.0 | Accepted |
| [0004](0004-multi-tenant.md) | Multi-tenant SaaS, with row-level isolation | Accepted |
| [0005](0005-accounting-as-plugin.md) | Accounting systems behind a plugin interface | Accepted |
| [0006](0006-configuration-over-customisation.md) | Shop-specific behaviour is configuration, not code | Accepted |
| [0007](0007-backend-dotnet.md) | Back end: .NET and ASP.NET Core | Accepted |
| [0008](0008-database-postgresql.md) | Database: PostgreSQL | Accepted |
| [0009](0009-frontend-react.md) | Front end: React, TypeScript and Mantine | Accepted |
| [0010](0010-api-first.md) | API first: one API for every client and every actor | Accepted |
| [0011](0011-documentation-first.md) | Documentation before implementation | Accepted |
| [0012](0012-testing.md) | Testing strategy | Accepted |
| [0013](0013-work-in-the-open.md) | Work in the open | Accepted |
| [0014](0014-build-journal.md) | Build journal | Accepted |
| [0015](0015-separate-deployments-one-origin.md) | Separate deployments for UI and API, served from one origin | Accepted |
| [0016](0016-rql-queries.md) | API queries use RQL | Accepted |
| [0017](0017-object-ids.md) | Object IDs | Accepted |
| [0018](0018-soft-delete.md) | Soft deletion by default | Accepted |
| [0019](0019-audit-trail.md) | Audit trail | Proposed |
| [0020](0020-url-structure.md) | URL structure | Accepted |
| [0021](0021-authentication.md) | Authentication | Accepted |
| [0022](0022-authorisation-and-roles.md) | Authorisation and roles | Accepted |
| [0023](0023-email-sending.md) | Email sending | Accepted |
| [0024](0024-product-analytics.md) | Product analytics | Accepted |
| [0025](0025-observability.md) | Observability: logs, traces, metrics and alerts | Accepted |
| [0026](0026-modular-monolith.md) | Modular monolith | Accepted |
| [0027](0027-file-storage.md) | File storage | Accepted |
| [0028](0028-branching-ci-and-releases.md) | Branching, CI and releases | Accepted |
| [0029](0029-api-errors-idempotency-versioning.md) | API errors, idempotency and versioning | Accepted |
| [0030](0030-hosting-aws.md) | Hosting: AWS in Sydney | Accepted |
| [0031](0031-concurrency.md) | Concurrency | Accepted |
| [0032](0032-events-webhooks-live-updates.md) | Events, webhooks and live updates | Accepted |
| [0033](0033-feature-flags.md) | Feature flags | Accepted |
| [0034](0034-background-work.md) | Background work | Accepted |
| [0035](0035-retries-and-resilience.md) | Retries and resilience | Accepted |
| [0036](0036-money-tax-and-time.md) | Money, tax, time and quantities | Accepted |

New records use the [ADR template](https://github.com/signboard-works/signboard/blob/main/docs/engineering/templates/adr-template.md).
