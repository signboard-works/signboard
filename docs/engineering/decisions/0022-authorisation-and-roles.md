# 0022. Authorisation and roles

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Access comes from memberships of accounts, and every request acts as one membership ([ADR 0010](0010-api-first.md)). One central policy decides which records and which fields an actor sees, for responses, RQL queries and audit snapshots alike ([ADR 0010](0010-api-first.md), [ADR 0016](0016-rql-queries.md), [ADR 0019](0019-audit-trail.md)). Operations users have no access to a business's data unless they add themselves to its account ([ADR 0010](0010-api-first.md)). API clients belong to one account and hold scoped tokens ([ADR 0021](0021-authentication.md)).

This record decides how permissions and roles work within that frame. In a small shop, people wear several hats: the owner sells, designs and quotes. Some data is sensitive within a business, not just between businesses: production staff may not need to see prices, and customers must never see costs or margins.

## Decision

### Permissions

- A **permission** is a named right to do one thing, by module and action, such as `sales.quotes.read`, `sales.quotes.send`, `production.jobs.move` or `billing.invoices.create`. Modules match the API's modules ([ADR 0020](0020-url-structure.md)).
- Permissions are defined in code, because each one needs code behind it to enforce it. The full list is kept in the authorisation design.
- **Deny by default.** Every endpoint and action declares the permission it needs. An automated test fails if any endpoint declares none.
- Permissions are worked out on the server for every request, from the membership's roles. They are never stored in the session cookie or in a token, so a role change takes effect on the next request.

### Field groups

- Every field belongs to one **field group**, such as `pricing.cost` (cost prices, margins, supplier prices), `pricing.sell` (sell prices and totals), `customer.contact` or `internal.notes`. Fields with no particular sensitivity belong to the object's general group.
- A permission such as `fields.pricing.cost.read` grants visibility of a group. A field the actor cannot see is omitted from responses, cannot be filtered, sorted or selected, and is stripped from audit snapshots.
- Permission to change a field requires permission to see it.

### Roles

- A **role** is a named set of permissions and field groups.
- **Built-in roles only, for now.** Signboard ships the roles below, and businesses cannot yet create their own. Custom roles are a later decision.
- **A membership can hold several roles.** Its permissions are the combination of all of them.

Built-in roles by account kind:

| Account kind | Role | In short |
|---|---|---|
| Business | **Administrator** | Everything, including settings, members, roles and API clients |
| Business | **Sales** | Customers, quotes and sales orders in full, including costs and margins; reads jobs and invoices |
| Business | **Designer** | Reads quotes and orders; works on design and, where used, proofs; no cost or margin |
| Business | **Production** | Reads and moves jobs through work centres; no prices |
| Business | **Field staff** | Reads jobs and their site-visit details, such as an installation's, and records progress on site; no prices. Offered when the business uses site visits ([ADR 0049](0049-scope-made-to-order-businesses.md)) |
| Business | **Accounts** | Invoices, payments and accounting sync in full; reads customers, quotes and orders, including costs |
| Business | **Signboard support** | Given to an operations user who adds themselves to the account: reads everything, changes nothing |
| Customer | **Customer administrator** | Everything the customer account can see and do, including managing its users |
| Customer | **Approver** | Views, and approves or declines, quotes and, where used, proofs |
| Customer | **Viewer** | Views only |
| Operations | **Platform administrator** | Runs the platform, including operations memberships |
| Operations | **Support** | Supports businesses, including adding themselves to a business's account |
| Operations | **Read-only** | Views platform data only |

The exact permissions in each role are set out in the authorisation design. The table above is their intent.

### Record scope

- Business staff see all of their business's records, limited by their permissions and field groups.
- Customer users see only their own customer account's records.
- Narrower scopes (for example, field staff seeing only jobs assigned to them) are a later decision.

### Managing memberships

- Business administrators invite users to the business by email and assign their roles.
- Customer users are managed by **both** the business (staff with the right permission) and the customer's own administrators. The business can always override.
- An operations user who adds themselves to a business's account gets the **Signboard support** role. If they need to change something, either the business grants more, or the operations user changes their own role. A role change made by an operations user is notified to the business's administrators and audited, exactly like adding themselves ([ADR 0010](0010-api-first.md)).

### Guard rails

- A business always has at least one Administrator. Removing or demoting the last one is refused.
- Nobody can grant a permission or role they do not hold themselves.
- An API token's permissions cannot exceed those of the person who created it.
- Every change to memberships, roles and API tokens is audited ([ADR 0019](0019-audit-trail.md)).

### Deferred

- Custom roles defined by a business.
- Narrower record scopes.
- Limits and approvals, such as maximum discounts or approval thresholds. These are conditions on actions, to be decided with business rules.

## Options considered

1. **Code-defined permissions, built-in roles, several roles per membership, field groups**: simple to understand and test; covers a small shop with no configuration; leaves room for custom roles. **Chosen.**
2. **Custom roles from the start**: maximum flexibility; needs a role editor and careful testing of arbitrary combinations before anyone needs them.
3. **One role per membership**: simplest model; forces a custom role for anyone who does more than one job, which is most people in a small shop.
4. **Permissions per individual field**: precise; hundreds of switches nobody can manage.
5. **Attribute-based policies** (rules over arbitrary attributes): very expressive; hard to reason about, test and explain to a shop owner.

## Consequences

- The authorisation design lists every permission and field group, and the exact contents of each built-in role. It must be written before the first endpoint.
- Every field in every object's design is assigned a field group.
- Every endpoint is tested for each built-in role: allowed, refused, and fields hidden ([ADR 0012](0012-testing.md)).
- Adding custom roles later needs only an editor and storage for role definitions, because roles are already sets of permissions.
