# 0010. API first: one API for every client and every actor

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard has several kinds of client: the shop's own web app, customers approving quotes and proofs, and integrations (accounting sync, scripts, other systems a shop already uses). A common pattern is a private API for the UI plus a separate, smaller public API for integrations. The two drift apart: the public API lags behind, has different rules, and doubles the work of testing and documenting.

Different actors must see different data. A customer must never see cost prices, margins or internal notes. A production staff member may not need to see pricing at all.

People do not fit neatly into one shop. A designer may work for two shops. The office manager of a club may approve quotes from one shop and also order from another. A sign shop may itself be a customer of another shop. Identity therefore cannot belong to a shop.

## Decision

Signboard has **one API**. It is the only way to read or change data, for every client and every actor.

- **No private endpoints.** The web app uses only the public, documented API. Anything the web app can do, an integration with the right permissions can do.
- **One surface for all actors.** Business staff, customer users in the customer portal and integration clients call the same endpoints. There is no separate customer API or partner API.
- **Users are global.** The system holds one set of users, each identified by a unique email address. A user is not owned by any business.
- **Access comes from memberships of accounts.** There are three kinds of account:
    - A **business account** is one sign or print shop, called a *business* in Signboard because "shop" suggests an online store. It is the tenant ([ADR 0004](0004-multi-tenant.md)). Its members are the business's staff, each with a role.
    - A **customer account** is one of a business's customers, and belongs to exactly one business. Its members use that business's customer portal. An organisation that buys from two businesses is two customer accounts, one in each business, and the same user can be a member of both.
    - The **operations account** is the platform itself. Its members run the hosted service.

    Further kinds can be added without changing the model; a **supplier account**, for a business's suppliers, is a possibility. A user can be a member of any number of accounts of any kind. A user with no memberships can sign in but sees nothing.
- **No account implies access to another.** Membership of the operations account gives no access to any business or customer account. Operations users can add themselves as members of another account when they need to, but the account's administrators are notified by email when it happens, the membership is visible to them and recorded in the audit log, and they can remove it like any other. It does not expire on its own.
- **Every request acts as one membership.** A request is made by a user acting in one account. The actor is that pair: the user plus the membership they are acting in. The membership decides the tenant ([ADR 0004](0004-multi-tenant.md)), the records in scope and the fields visible. A user who belongs to several businesses and customers switches between them; they never see several at once in a single request.
- **One surface, more than one way to sign in.** The web app authenticates with a secure, HttpOnly session cookie; integration clients use bearer tokens. Both resolve to the same actor and reach the same endpoints under the same policy. Only the proof of identity differs, never the API.
- **Integration clients are members too.** An API client belongs to one business or customer account, with scoped permissions, and is subject to exactly the same policy as a user acting in that membership.
- **Actor-scoped visibility.** Authorisation decides which records an actor can see (a customer user sees only their own customer's quotes, orders and invoices). It also decides which fields of those records they can see. Fields an actor may not see are omitted from the response, and the OpenAPI contract documents the visibility of each field per account kind and role. Attempting to write a field the actor may not change is rejected with an error, never silently ignored.
- **Central policy.** Record scope and field visibility are defined in one policy layer and applied to every endpoint. They are never rewritten ad hoc in individual handlers.
- **Contract first.** The API is resource-oriented JSON over HTTPS, described by an OpenAPI 3.1 contract. The contract for a feature is written and reviewed as part of its design doc, before the endpoints are implemented ([ADR 0011](0011-documentation-first.md)).
- **One query language.** Collection endpoints are filtered, sorted, projected and paged with RQL, and field visibility applies to queries as well as responses ([ADR 0016](0016-rql-queries.md)).
- **Versioned.** Endpoints live under `/api/v1`. Additive changes are made within a version. Breaking changes need a new major version and a deprecation period.

## Options considered

1. **One API for every client and actor, with actor-scoped visibility**: one thing to build, document, secure and test; integrations are never second-class. **Chosen.**
2. **Private UI API plus public integration API**: lets the UI move fast; the two drift apart and the public API is always incomplete.
3. **Backend-for-frontend layer**: endpoints shaped for each screen; convenient, but it is a private API under another name.
4. **GraphQL**: flexible querying for the UI; field-level authorisation is possible but harder to reason about and cache, and REST is more familiar to integrators.

## Consequences

- The field-level authorisation layer is foundational and must be designed in full before the first endpoint ([vision: plan the foundations](../product/vision.md)).
- Every endpoint is tested as each account kind and role, including that hidden fields are absent ([ADR 0012](0012-testing.md)). Tests also cover a user with memberships in two businesses, to prove that acting in one never exposes the other.
- Users and the index of their memberships are platform-level data, outside any tenant, so a user can list the accounts they belong to. Everything a membership gives access to stays inside its tenant.
- Email addresses are identities, so changing one, verifying one and handling a person who leaves a business or customer need careful design (removing the membership, not the user).
- Screens that need combined data get a public, documented endpoint designed for that need, rather than a private shortcut.
- Customer-facing features (quote and proof approval, viewing invoices) need customer users to be first-class authenticated actors, not just recipients of emails. Each business has its own customer portal, at the business's subdomain, in scope for phase 2. Custom domains for businesses are a later, separate decision.
- The API documentation is a product in its own right and is published with the docs site.
