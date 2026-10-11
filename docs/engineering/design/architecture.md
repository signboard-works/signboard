# Design: Architecture

| | |
|---|---|
| **Status** | Draft |
| **Author** | Stuart Meeks |
| **Last updated** | 2026-10-10 |
| **Phase** | 1 |

## Context

The overall shape of the system: components, boundaries, and how they talk.

## Known constraints (to be expanded)

- Modular monolith: one codebase and one database; modules (`accounts`, `catalog`, `sales`, `production`, `billing`, `notifications`, `system`) with boundaries enforced by architecture tests; deployables are the UI, API, background worker and identity service ([ADR 0026](../decisions/0026-modular-monolith.md)).
- Multi-tenant SaaS: shared database and schema, tenant isolation by PostgreSQL row-level security and EF Core query filters; also self-hostable as a single tenant ([ADR 0004](../decisions/0004-multi-tenant.md)).
- Accounting behind a plugin interface ([ADR 0005](../decisions/0005-accounting-as-plugin.md)).
- One API for every client and actor, with actor-scoped record and field visibility ([ADR 0010](../decisions/0010-api-first.md)).
- Back end: .NET LTS and ASP.NET Core ([ADR 0007](../decisions/0007-backend-dotnet.md)); PostgreSQL via EF Core ([ADR 0008](../decisions/0008-database-postgresql.md)).
- Front end: React and TypeScript single-page app ([ADR 0009](../decisions/0009-frontend-react.md)).
- UI and API deployed separately, served from one origin per business by an edge router ([ADR 0015](../decisions/0015-separate-deployments-one-origin.md)).
- Hosted on AWS in Sydney, with every data store in Australia; every AWS service has a self-hosted equivalent ([ADR 0030](../decisions/0030-hosting-aws.md)).

## Design

_To do._

## Failure modes

_To do._

## Alternatives considered

_To do._

## Open questions

_To do._
