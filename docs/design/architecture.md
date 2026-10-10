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

- Multi-tenant SaaS: shared database and schema, tenant isolation by PostgreSQL row-level security and EF Core query filters; also self-hostable as a single tenant ([ADR 0004](../adr/0004-multi-tenant.md)).
- Accounting behind a plugin interface ([ADR 0005](../adr/0005-accounting-as-plugin.md)).
- One API for every client and actor, with actor-scoped record and field visibility ([ADR 0010](../adr/0010-api-first.md)).
- Back end: .NET LTS and ASP.NET Core ([ADR 0007](../adr/0007-backend-dotnet.md)); PostgreSQL via EF Core ([ADR 0008](../adr/0008-database-postgresql.md)).
- Front end: React and TypeScript single-page app ([ADR 0009](../adr/0009-frontend-react.md)).
- UI and API deployed separately, served from one origin per business by an edge router ([ADR 0015](../adr/0015-separate-deployments-one-origin.md)).

## Design

_To do._

## Failure modes

_To do._

## Alternatives considered

_To do._

## Open questions

_To do._
