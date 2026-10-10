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

New records use the [ADR template](https://github.com/signboard-works/signboard/blob/main/docs/templates/adr-template.md).
