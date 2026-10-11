# 0008. Database: PostgreSQL

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard is a multi-tenant service whose tenants share one database, with isolation enforced by row-level security ([ADR 0004](0004-multi-tenant.md)). The same code must also be free for a shop to self-host. The database must be free to run in production at any size, enforce row-level security natively, be easy to back up, and licensed compatibly with an Apache 2.0 project ([ADR 0003](0003-licence.md)). The maintainer's background is Microsoft SQL Server.

## Decision

We will use PostgreSQL (current supported major version) as the only supported database, accessed through Entity Framework Core with the Npgsql provider. Schema changes are EF Core migrations, applied automatically on upgrade.

## Options considered

1. **PostgreSQL**: open source, free at any size, mature row-level security, small footprint, mature tooling for backup and restore, excellent EF Core support, JSON columns for flexible configuration. **Chosen.**
2. **Microsoft SQL Server**: the maintainer's strongest database skill. Proprietary; the free Express edition caps database size and memory; licence costs for the hosted service scale with cores; the Linux container needs at least 2 GB of RAM, a burden for self-hosters.
3. **Support both**: maximum choice for self-hosters; every migration and query tested twice, for little real benefit.
4. **SQLite**: zero setup; weak concurrency and limited types for a multi-user business system.

## Consequences

- The maintainer learns PostgreSQL specifics (types, indexing, `pg_dump`). Most SQL and EF Core knowledge carries over.
- Tests run against real PostgreSQL in containers, never an in-memory substitute ([ADR 0012](0012-testing.md)).
- Provider-specific features (row-level security, JSON columns, full-text search) can be used freely, because there is only one provider.
