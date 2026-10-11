# 0004. Multi-tenant SaaS, with row-level isolation

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard is built first for one shop and then offered to any shop that wants it. Most small shops do not want to run a server, back it up or upgrade it. They want to sign up and use it. The maintainer intends to offer Signboard as a hosted service, possibly a for-profit one, and the licence allows this ([ADR 0003](0003-licence.md)).

Running one instance per business keeps the data model simple, but every business then costs a container, a database, a backup schedule and an upgrade. That works for dozens of businesses and becomes an operations job beyond that. Adding tenancy to a system that already holds live data is far more expensive than designing it in from the start, and the [vision](../../vision.md) says foundations are designed in full, up front.

The worst possible failure of a multi-tenant system is one business seeing another business's customers, prices or invoices. Isolation must not depend on every query being written correctly.

## Decision

Signboard is a multi-tenant application. A **tenant** is one **business**: a sign or print shop and everything it holds, including its customers.

- **Shared database, shared schema.** Every tenant-owned table has a non-null `tenant_id`. Only platform-level data (tenants, users, the operations account, the index of users' memberships, plans) sits outside a tenant.
- **Isolation enforced twice.** PostgreSQL row-level security policies on every tenant-owned table restrict rows to the current tenant, which the application sets for each transaction. The application connects as a role that does not own the tables and cannot bypass row-level security. EF Core global query filters apply the same rule in the application, so a mistake in one layer is caught by the other.
- **Tenant from the membership, checked on every request.** Users are global and may belong to several business and customer accounts ([ADR 0010](0010-api-first.md)). Each request acts as one membership, and the tenant is the business that account belongs to. The web app and customer portal identify the business from the subdomain (`<business>.signboard.works`), and the request is refused unless the user holds an active membership of that business's account or one of its customer accounts. Each business has its own customer portal at its subdomain; custom domains for businesses are a later, separate decision. API clients belong to one business or customer account, so they always act in that tenant.
- **Tenant context everywhere, not just the database.** Background jobs, file storage paths, cache keys, search indexes, logs and outbound email all carry the tenant.
- **Operations users are not business members.** Platform operators are members of the operations account ([ADR 0010](0010-api-first.md)), which gives no access to any tenant's data. To give support, an operations user adds themselves as a member of the business's account. The business's administrators are notified by email when it happens, and the membership is visible to the business, recorded in the audit log and removable by the business. It does not expire on its own.
- **One codebase, still self-hostable.** The open-source code is the same code that runs the hosted service. A self-hosted installation is the same application with one tenant. There is no separate edition.
- **The business owns its data.** Each tenant can export all of its data in open formats at any time, and a tenant's data can be deleted completely when it leaves.

## Options considered

1. **Single tenant, one instance per business**: simplest data model; isolation by construction. Operating cost grows linearly with businesses, and a later move to multi-tenancy would mean reworking a live system.
2. **Multi-tenant, shared schema, row-level security plus query filters**: cheapest to run per business, one migration path, one deployment; isolation is enforced by the database as well as the application. **Chosen.**
3. **Multi-tenant, schema per tenant**: clearer separation; migrations run once per tenant, connection pooling gets harder, and EF Core supports it poorly.
4. **Multi-tenant, database per tenant**: strongest isolation and easy per-tenant restore; operating cost close to option 1. Could be offered later as a premium tier for a tenant that needs it, without changing the data model.
5. **Single tenant now, tenant-ready schema**: a `tenant_id` everywhere but no SaaS. Carries most of the cost of option 2 for none of the benefit.

## Consequences

- Every API test includes a cross-tenant case: an actor from tenant A gets nothing from tenant B ([ADR 0012](0012-testing.md)). An automated test fails if any tenant-owned table lacks a `tenant_id` or a row-level security policy.
- A user belongs to no tenant, but every request they make runs inside exactly one ([ADR 0010](0010-api-first.md)). A user who is a member of two businesses is the strongest cross-tenant test case, and is tested explicitly.
- Configuration is stored per tenant ([ADR 0006](0006-configuration-over-customisation.md)), which makes "configuration, not code" non-negotiable: one schema and one codebase serve every business.
- Restoring a single tenant from a shared database is harder than restoring a whole database. Per-tenant export and import tooling is needed before the first paying business.
- Per-tenant rate limiting is needed so one business cannot slow down the rest.
- Accounting connections ([ADR 0005](0005-accounting-as-plugin.md)) and email sending domains are configured per tenant, with secrets encrypted per tenant.
- Running a service brings obligations a self-hosted app does not: uptime, monitoring, backups, security response, and privacy obligations under the Australian Privacy Act for each business's customer data. On a weekends-only project these need deliberate design, not good intentions.
- Sign-up, plans and billing are separate decisions, to be made in their own ADR and PRD.
