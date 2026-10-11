# 0026. Modular monolith

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard is built and run by one part-time maintainer. Its core workflow, quote to sales order to job to invoice, is a chain in which one action changes several areas at once: accepting a quote creates a sales order and jobs.

Several decisions already assume one database:

- tenant isolation by row-level security ([ADR 0004](0004-multi-tenant.md));
- RQL queries across related objects ([ADR 0016](0016-rql-queries.md));
- audit triggers on every table ([ADR 0019](0019-audit-trail.md));
- the email outbox written in the same transaction as the change that causes it ([ADR 0023](0023-email-sending.md)).

Microservices solve an organisational problem: many teams that need to build and deploy without coordinating. Each service adds a pipeline, a deployment, monitoring, version compatibility and failures between services. A monolith avoids all of that, but without firm internal boundaries it decays into code where everything depends on everything.

## Decision

Signboard is a **modular monolith**: one codebase and one database, divided into modules with enforced boundaries.

### Modules

- The modules are the API's modules ([ADR 0020](0020-url-structure.md)): `accounts`, `catalog`, `sales`, `production`, `billing`, `notifications` and `system`.
- Each module is its own .NET project, owns its own tables, and exposes a public interface to other modules.
- **A module never reads or writes another module's tables.** It calls the other module's interface.
- **Boundaries are enforced by architecture tests** (with a library such as ArchUnitNET or NetArchTest). A dependency that crosses a boundary any other way fails the build ([ADR 0012](0012-testing.md)).
- Work that spans modules, such as accepting a quote, runs in **one database transaction**, coordinated through the modules' interfaces.
- Work that can happen afterwards, such as notifying people, reacts to another module's business events ([ADR 0032](0032-events-webhooks-live-updates.md)).

### Deployables

One codebase produces a few deployables:

| Deployable | Role |
|---|---|
| UI | Static files behind a CDN ([ADR 0015](0015-separate-deployments-one-origin.md)) |
| API | Serves `/api` on every business's host |
| Background worker | Email sending, accounting sync, imports and other long-running jobs. Same code and image as the API with a different start-up, so it scales and restarts independently |
| Identity service | `id.signboard.works` ([ADR 0021](0021-authentication.md)). Kept separate so security-critical code is smaller, easier to review and easier to lock down |

All of them share the one PostgreSQL database, each with its own database role and only the permissions it needs.

### When to split a service out

A separate service is justified only by a concrete need that the monolith cannot meet, for example:

- a part with radically different technical needs, such as heavy processing of artwork or print files;
- a capability that must be isolated for security or compliance, such as payments;
- several teams that need to deploy independently.

Splitting one out needs its own ADR. Because modules already talk only through interfaces, extracting one is contained work rather than a rewrite.

## Options considered

1. **Modular monolith with a few deployables**: one transaction for cross-module work, one database for every earlier decision that relies on it, one pipeline to run; boundaries kept by tests. **Chosen.**
2. **Microservices**: independent deployment and scaling per service; distributed transactions (sagas) for the core workflow, and a large operational load for a single maintainer.
3. **Plain monolith without enforced modules**: least structure to set up; boundaries erode, and any later split becomes a rewrite.
4. **One process for everything**: simplest deployment; long-running jobs compete with requests, and the identity service shares its attack surface with the whole API.

## Consequences

- The module list and each module's public interface are set out in the architecture design, before the first module is built.
- Architecture tests are part of the build from the first commit.
- Every deployable is built from the same codebase and released together, so there is no version compatibility between them to manage.
- The database is a single shared dependency. Its availability, backups and performance are the service's, and are designed with hosting.
