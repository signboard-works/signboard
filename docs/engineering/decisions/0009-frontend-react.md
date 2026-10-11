# 0009. Front end: React, TypeScript and Mantine

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

The front end is one client of the public API ([ADR 0010](0010-api-first.md)). The maintainer is not a front-end specialist, so a large ecosystem of well-documented components matters more than keeping one language across the stack.

Three kinds of view are known to be demanding:

- **Grids.** Most screens are lists: quotes, sales orders, jobs, invoices, customers. Filtering, sorting and paging all happen on the server through RQL ([ADR 0016](0016-rql-queries.md)), and users want their own column layouts to persist.
- **The production kanban board.** Jobs are dragged between work centres, often on a tablet on the shop floor.
- **A calendar**, later, for scheduling installs and due dates, possibly by crew.

Any library used must have a licence compatible with Apache 2.0 ([ADR 0003](0003-licence.md)), and must not put core features behind a paid tier.

## Decision

We will build the front end as a single-page application in React and TypeScript, built with Vite into static files. It runs entirely in the browser and reaches data only through the public API. How it is deployed and served is decided in [ADR 0015](0015-separate-deployments-one-origin.md).

- **One app for every account kind.** The business's staff UI and the customer portal are the same application, showing different screens and navigation by account kind. Routes are code-split, so customer users download only what the portal needs.
- **Component library: Mantine** (MIT). We accept that Mantine depends mostly on one maintainer, and contain that risk with the wrapper rule below.
- **Wrapper rule.** Screens use Signboard's own components (such as `DataGrid`, `FormField` and `PageLayout`), never Mantine directly. Only the wrapper components import Mantine, so replacing it would be a contained job rather than a rewrite of every screen.
- **One `DataGrid`**, built on TanStack Table (MIT) and rendered with Mantine components, used for every list in the app. It:
    - runs in server mode: every change to filters, sorting or the page becomes an RQL request, and paging uses the API's `$meta.pagination`;
    - builds its filter controls from the API's per-actor query metadata, so it never offers a filter the API would refuse;
    - keeps the current query in the page URL, so a filtered, sorted view is a shareable link;
    - remembers each user's **table view**: visible columns, column order, column widths, pinned columns, sort, filters, page size and density. Table views belong to the user, not the business, and are stored in the browser, so they do not follow the user to another device;
    - starts from the business's **default view** for each grid when the user has none of their own. A business administrator sets default views, which are stored through the API. "Reset to default" discards the user's own view;
    - works by touch.
- **Kanban board:** built on a drag-and-drop library, chosen in a spike between dnd-kit (MIT) and Atlassian's Pragmatic drag and drop (Apache 2.0). **Drag and drop must work by touch on a tablet.** This is a hard requirement.
- **Calendar:** chosen in the scheduling design doc. Candidates include FullCalendar's MIT-licensed standard edition and react-big-calendar (MIT). FullCalendar's per-resource (per-crew) views are a paid edition, so the choice waits until we know whether installs are scheduled by crew.
- **API client:** generated from the OpenAPI contract, never hand-written.
- **Accessibility:** WCAG 2.2 AA, including keyboard alternatives to every drag-and-drop action.

Data fetching, routing, forms, state, formatting and the design of table views are covered in the [front-end design doc](../design/front-end.md).

## Options considered

### Component library

1. **Mantine**: broad, well-documented and entirely free; simpler to theme than MUI; mostly one maintainer. **Chosen.**
2. **MUI with MUI X**: the most widely used React library, backed by a company; the advanced data grid, date range pickers and scheduler are partly under paid licences.
3. **shadcn/ui (Radix and Tailwind)**: components copied into the repository and fully owned; more assembly and styling work, which suits front-end specialists more than this project.
4. **Blazor with MudBlazor**: one language (C#) across the stack; smaller ecosystem, especially for kanban and calendar components. Rejected in favour of the React ecosystem.
5. **Next.js**: server rendering is unnecessary for a back-office app, and it adds a Node.js server to every deployment.

### Grid

1. **TanStack Table with our own Mantine rendering**: headless, actively maintained and widely used. Most of the grid's furniture (RQL translation, metadata-driven filters, URL state, table views) is Signboard's own code with any library. **Chosen.**
2. **mantine-datatable**: MIT, tracks current Mantine, supports server-side paging and sorting; a better-looking table on day one, but a second single-maintainer dependency for little gain.
3. **Mantine React Table**: rejected. Its version 2 has been in beta since February 2025 and targets Mantine 7, while Mantine is at version 9.
4. **MUI X Data Grid**: the most capable grid, but much of its value is client-side features Signboard does not use, and the rest is partly paid.

## Consequences

- Two languages and two build toolchains (dotnet and npm) in one repository.
- The wrapper components and `DataGrid` are foundational and are designed before the first screen.
- Business default views need an API resource of their own. Users' table views need none, because they stay in the browser.
- A user's table views are lost if they clear their browser data, and do not exist on a new device until they arrange the grid again. Business default views soften this.
- The kanban board and calendar each need a short spike before their design docs are approved.
- Mantine's major versions bring breaking changes. The wrapper rule keeps upgrades contained, and they should be done promptly.
