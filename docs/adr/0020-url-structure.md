# 0020. URL structure

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard's URLs are part of its public contract. Integrators code against API paths ([ADR 0010](0010-api-first.md)), emails to customers link to quotes and invoices, and staff bookmark and share filtered grids ([ADR 0016](0016-rql-queries.md)). A URL that changes breaks all of these, so the rules are set before the first endpoint or screen.

Each business has its own subdomain, serving both the app and the API from one origin ([ADR 0004](0004-multi-tenant.md), [ADR 0015](0015-separate-deployments-one-origin.md)). Every object has one ID ([ADR 0017](0017-object-ids.md)).

## Decision

### Hosts

| Host | Serves |
|---|---|
| `<business>.signboard.works` | The business's staff app, its customer portal, and the API at `/api` |
| `signboard.works` | The public website |
| `docs.signboard.works` | This documentation, including the API reference |
| `ops.signboard.works` | The operations account's app |

- A business's **subdomain name** is lowercase letters, digits and hyphens, 3 to 63 characters, starting with a letter. It is chosen at sign-up and unique across Signboard.
- If a business's subdomain name changes, the old name redirects to the new one and is never given to another business.
- Names for Signboard's own use are reserved and can never be a business's: for example `www`, `api`, `app`, `docs`, `ops`, `id`, `status`, `mail`, `admin`, `support`.
- Custom domains for businesses are a later decision ([ADR 0004](0004-multi-tenant.md)). The paths below are the same on any host.

### API paths

```
/api/v1/{module}/{collection}                              list (RQL) or create
/api/v1/{module}/{collection}/{id}                         one object
/api/v1/{module}/{collection}/{id}/{child-collection}      objects that exist only within their parent
/api/v1/{module}/{collection}/{id}/{action}                a state change that is not a plain edit
```

Examples:

```
GET    /api/v1/production/jobs?and(eq(status,in_production),ge(dueDate,2026-11-01))&order=-dueDate&limit=50
GET    /api/v1/sales/quotes/QUO-4821-0937-1156
GET    /api/v1/sales/quotes/QUO-4821-0937-1156/versions/2
POST   /api/v1/sales/quotes/QUO-4821-0937-1156/send
PATCH  /api/v1/accounts/accounts/ACC-4821-0937
DELETE /api/v1/production/jobs/JOB-4821-0937-1156
```

- **Modules group collections.** Every collection belongs to exactly one module, and the module is the first path segment after the version. The initial modules are:

    | Module | Holds, for example |
    |---|---|
    | `accounts` | Accounts (business, customer, operations), users, memberships, API clients |
    | `catalog` | Products, materials, pricing rules |
    | `sales` | Quotes, sales orders |
    | `production` | Jobs, work centres |
    | `billing` | Invoices, payments |
    | `notifications` | Emails and their delivery events |
    | `system` | Audit trail, settings |

    The full list of modules and their collections is kept in the API design. Adding a module is a design decision; moving a collection between modules is a breaking change.
- **Module and collection names are lowercase kebab-case, collections plural nouns:** `/sales/quotes`, `/sales/sales-orders`, `/production/work-centres`.
- **Objects are addressed by their object ID** ([ADR 0017](0017-object-ids.md)). There are no slugs or alternative keys in paths. A versioned object's versions are numbered beneath it (`/versions/2`).
- **Nesting** is used only for objects that cannot exist without their parent, such as a quote's versions and their lines, and goes no more than two levels deep. An object that exists in its own right has its own top-level collection, even when it is usually reached from another.
- **Methods:** `GET` reads, `POST` on a collection creates, `PATCH` updates with JSON Merge Patch (RFC 7396), `DELETE` soft-deletes ([ADR 0018](0018-soft-delete.md)). There is no `PUT`.
- **Actions** are `POST` to a verb beneath the object, for state changes with rules or side effects that a field update cannot express: `/send`, `/approve`, `/revise`, `/restore`.
- **Query strings** carry RQL and nothing else ([ADR 0016](0016-rql-queries.md)).
- **No account in the path.** The account a request acts in comes from its credentials, not its URL: an API client belongs to one account, and a browser session holds the account the user is currently acting in ([ADR 0010](0010-api-first.md)).
- **Exact paths:** lowercase, no trailing slash, no file extensions. The API does not redirect; a path that does not match exactly is `404 Not Found`.
- **Field names** in JSON bodies and in RQL are camelCase, and are the same in both.

### Responses

Every request on a single object returns **the full object** as its response body, in its state after the request, so a client never needs to fetch it again after a write:

| Request | Status | Body |
|---|---|---|
| `GET` an object | `200 OK` | The object |
| `POST` to a collection (create) | `201 Created` | The new object, including its ID ([ADR 0017](0017-object-ids.md)) |
| `PATCH` an object | `200 OK` | The object after the update |
| `POST` an action (`/send`, `/approve`, …) | `200 OK` | The object after the action |
| `DELETE` an object | `200 OK` | The soft-deleted object, with `deletedAt` and `deletedBy` set ([ADR 0018](0018-soft-delete.md)) |

- **Full means the whole representation**, not only the fields that changed or were sent. An object's representation, including which dependent child objects it embeds, is defined in that object's design.
- **Field visibility still applies** ([ADR 0010](0010-api-first.md)): the response omits every field the actor may not see, exactly as a `GET` would. Writing an object never reveals more of it.
- The response is the same representation a `GET` immediately afterwards would return.
- Collection requests return objects inside the paging envelope, and RQL `select` may narrow them ([ADR 0016](0016-rql-queries.md)).

### App paths

App URLs mirror the API's modules, collections and object IDs, so a link means the same thing in both:

```
/sales/quotes                                   list, with the grid's RQL query in the URL
/sales/quotes/QUO-4821-0937-1156                one quote
/production/jobs/board                          the production kanban board
/system/settings/...                            business settings
```

- **One URL per object, for every actor.** Business staff and customer users open the same URL for the same object. What the app shows there depends on the account the user is acting in ([ADR 0010](0010-api-first.md)): the same screen with fewer fields, or a different screen altogether. The customer portal is therefore not a separate set of URLs, and the URL structure does not depend on how much the business and customer screens turn out to share. Links in emails work for everyone who may open them.
- App URLs follow the same rules as API paths: lowercase kebab-case, object IDs, no trailing slash. Unlike the API, the app tidies near-misses (a trailing slash, an ID in lowercase) by redirecting to the exact URL.
- App URLs that have appeared in an email or been bookmarked are treated as permanent. If one must change, the old URL redirects.

## Options considered

1. **Collections grouped by module, addressed by object ID, actions as verb sub-paths, account from credentials, one URL per object for every actor**: predictable URLs; related resources sit together, as in the Marketplace API that integrators may already know; links that survive. **Chosen.**
2. **Flat collections** (`/api/v1/quotes`): shorter paths; nothing groups related resources, and the API reference becomes one long list.
3. **Account in the path** (`/api/v1/accounts/{id}/quotes`): the acting account is visible in every URL; every path is longer, and the same object has a different URL for every account that can see it.
4. **Business in the path instead of a subdomain** (`signboard.works/acme/...`): one host; loses the per-business origin that cookies and custom domains rely on ([ADR 0015](0015-separate-deployments-one-origin.md)).
5. **`PUT` for updates**: widely used; full replacement makes partial updates awkward and risks wiping fields an actor cannot see.
6. **Minimal responses to writes** (`204 No Content`, an ID, or only the changed fields): smaller; every client must fetch again to see the result, and may see a later state than its own write produced.
7. **Optional full responses** (via a `Prefer: return=representation` header): flexible; two behaviours to document and test for every endpoint.
8. **A separate customer portal path or host** (`/portal/...` or `portal.<business>.signboard.works`): keeps customer screens apart; gives one object two URLs and fixes the split between business and customer screens before it is designed.

## Consequences

- API paths and app routes can be generated and checked from the same list of modules and collections.
- Every screen that both business staff and customer users can reach must handle both, by account kind.
- Every write ends by loading and serialising the object through the same path and field-visibility policy as `GET`. Tests check that a write's response equals a subsequent `GET`, for each actor ([ADR 0012](0012-testing.md)). Objects that embed large child collections need a deliberate representation in their design.
- Renaming a module or collection, or moving a collection between modules, is a breaking API change and needs a new API version ([ADR 0010](0010-api-first.md)).
- Subdomain names need a reserved-word list and a redirect table for renamed businesses.
- Where users sign in, given that users are global and each business has its own host, is a separate decision for the authentication design.
