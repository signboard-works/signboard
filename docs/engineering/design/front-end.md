# Design: Front end

| | |
|---|---|
| **Status** | Draft |
| **Author** | Stuart Meeks |
| **Last updated** | 2026-10-10 |
| **Implements** | Cross-cutting: every screen in every PRD |
| **Related ADRs** | [0009](../decisions/0009-frontend-react.md), [0010](../decisions/0010-api-first.md), [0015](../decisions/0015-separate-deployments-one-origin.md), [0016](../decisions/0016-rql-queries.md) |

## Context

How the React single-page application is structured: the shared components every screen is built from, how it talks to the API, and how it behaves for each kind of account.

## Known constraints (to be expanded)

- React and TypeScript, built with Vite; Mantine behind Signboard's own wrapper components ([ADR 0009](../decisions/0009-frontend-react.md)).
- Data only through the public API, using a client generated from the OpenAPI contract ([ADR 0010](../decisions/0010-api-first.md)).
- Signs in with a secure, HttpOnly session cookie on the same origin as the API ([ADR 0015](../decisions/0015-separate-deployments-one-origin.md)).
- One app for business staff and the customer portal, code-split by route.
- Must tolerate an API one release ahead, and prompt the user to reload when a newer UI is deployed.
- WCAG 2.2 AA. Drag and drop works by touch and has a keyboard alternative.

## Design

To be written. Areas to cover:

- **Data fetching and caching:** TanStack Query, with the generated API client.
- **Live updates:** one Server-Sent Events stream per tab; notifications invalidate the matching TanStack Query caches; grids refetch the visible page in place, detail pages refetch silently, forms being edited show a "changed by" banner, kanban cards animate to their new place ([ADR 0032](../decisions/0032-events-webhooks-live-updates.md)).
- **Routing:** React Router (or TanStack Router), with the RQL query in the URL for grids.
- **Forms and validation:** Mantine's form library behind the wrapper, with server-side validation errors mapped to fields.
- **Formatting:** dates, numbers, currency and tax from the business's settings ([ADR 0006](../decisions/0006-configuration-over-customisation.md)), never hard-coded. Times are shown in the business's time zone, labelled when the browser is elsewhere; the API calculates every amount, and on-screen previews use a decimal library ([ADR 0036](../decisions/0036-money-tax-and-time.md)). Interface text in English at first, structured so it can be translated later.
- **`DataGrid`:** grid state to and from RQL, metadata-driven filter controls, paging from `$meta.pagination`.
- **Table views:** see below.

### Table views

A table view is a grid's visible columns, column order, column widths, pinned columns, sort, filters, page size and density.

- **User table views** belong to the user, not the business, and are stored in the browser, keyed by user ID and grid, so several people sharing a tablet on the shop floor each keep their own. They do not follow the user to another device. Browser storage is separate for each origin and each business has its own subdomain, so in practice a user's table views are also separate in each business. That is accepted.
- **Business default views** are set by a business administrator for each grid and stored through the API. They can include filters. Active filters are always shown on the grid as visible, removable chips, so nobody misses that rows are hidden. A user with no table view of their own starts from the business default; "reset to default" discards the user's view.
- Saved automatically as the user changes the grid (debounced).
- On load, columns and filters that no longer exist, or that the actor may no longer see or filter by, are skipped, and the stored view is tidied. The API refuses such filters anyway ([ADR 0016](../decisions/0016-rql-queries.md)).
- Views carry a version number so their format can change without breaking stored views.

## Open questions

- [x] Table views separate per business in practice, because browser storage is per origin: accepted.
- [ ] Named views per grid (for example "My jobs due this week"): later, as a separate feature. One table view per user per grid for now.
- [ ] Router choice: React Router or TanStack Router.
