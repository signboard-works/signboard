# 0016. API queries use RQL

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Most of Signboard's screens are lists: quotes, sales orders, jobs, invoices, customers. Each needs filtering, sorting and paging, and so does every integration that reads the same data ([ADR 0010](0010-api-first.md)). Without a single query language, each endpoint invents its own parameters, every grid needs its own glue code, and integrators relearn the rules resource by resource.

Querying interacts with security. A customer user must not see a business's cost prices ([ADR 0010](0010-api-first.md)). If they could filter or sort by a hidden field, they could work out its value without it ever appearing in a response, for example by bisecting `gt(costPrice,…)` until the result changes.

Resource Query Language (RQL) is an established query-string syntax for filtering, sorting, selecting fields and paging. The SoftwareOne Marketplace API uses it, and its dialect is familiar to integrators who know that API. An open-source .NET implementation, [Mpt.Rql](https://github.com/softwareone-platform/mpt-rql-net), is licensed under Apache 2.0, actively maintained, and translates RQL into LINQ that Entity Framework Core runs in the database.

## Decision

Every collection endpoint accepts RQL in the Marketplace dialect, for example:

```
GET /api/v1/jobs?and(eq(status,in_production),ge(dueDate,2026-11-01))&order=-dueDate,number&select=customer&limit=50&offset=100
```

- **Implementation:** Mpt.Rql, applied to EF Core queries so filtering, sorting and projection run in PostgreSQL, beneath row-level security ([ADR 0004](0004-multi-tenant.md), [ADR 0008](0008-database-postgresql.md)).
- **Explicit allow-lists:** each resource declares, per property, whether it can be filtered, sorted and selected, and with which operators. Anything not declared is refused. Filterable and sortable properties must be backed by an index, or the design doc must say why one is not needed.
- **Field visibility applies to queries, not just responses:** for each request, the central policy layer ([ADR 0010](0010-api-first.md)) marks every field the actor may not see as hidden to RQL. A hidden field cannot be selected, filtered or sorted, and a query that refers to one is rejected.
- **Errors, never guesses:** an invalid query, a refused property or operator, or an out-of-range limit returns `400 Bad Request` with problem details listing each error. Nothing is silently dropped or adjusted.
- **Paging:** `limit` and `offset`. The default `limit` is 50 and the maximum is 1,000; a larger value is an error. The total number of matching records is always returned.
- **Stable order:** the record's unique ID is always appended as the final sort key, so paging never skips or repeats records.
- **Response shape:** collection responses use one envelope:

    ```json
    {
      "$meta": {
        "pagination": { "offset": 100, "limit": 50, "total": 1240 }
      },
      "data": [ ]
    }
    ```

- **Self-describing:** the API publishes, per resource and per actor, which fields can be filtered and sorted, their types and their allowed operators. Clients, including Signboard's own grids ([ADR 0009](0009-frontend-react.md)), build their filter controls from this, so they never offer a query the API would refuse. The mechanism (an OpenAPI extension or a metadata endpoint) is decided in the API design doc.

## Options considered

1. **RQL with Mpt.Rql**: one expressive, consistent query language across every resource; a maintained, Apache-licensed .NET library with per-property permissions and per-request visibility that match the field-visibility model. **Chosen.**
2. **RQL with our own parser**: full control; a parser, translator and validation suite to build and maintain for no gain over an existing library.
3. **OData**: a standard with strong .NET support; large and complex, its URL conventions are heavier for integrators, and fitting field-level visibility into it is harder.
4. **Ad hoc query parameters** (`?status=…&sort=…&page=…`): easy to start with; cannot express `or` or nested conditions, and drifts into a different dialect per endpoint.
5. **GraphQL**: already rejected in [ADR 0010](0010-api-first.md).

## Consequences

- Grids get one translator between grid state and RQL, used everywhere, and the RQL query can live in the page URL, so a filtered view is a shareable link.
- Every resource's integration tests cover its query allow-list, and, for each actor, that queries on hidden fields are rejected ([ADR 0012](0012-testing.md)).
- Always returning a total costs a count query per request. That is acceptable at Signboard's data volumes, but slow totals should be watched and indexed for.
- Signboard depends on a library maintained by a single company. It is open source under Apache 2.0, so if it were abandoned, Signboard could fork it.
- Mpt.Rql handles filtering, sorting and selection only. Paging, the total count and the response envelope are Signboard's own code, written once and shared by every endpoint.
- Sorting by a value inside a child collection (`first()`) runs a correlated subquery that cannot use an index. It is refused unless a resource's design doc allows it.
