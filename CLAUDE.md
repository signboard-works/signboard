# Signboard: working notes for Claude

Signboard is open-source, multi-tenant job management for sign and print shops: quotes, sales orders, jobs, a production kanban, invoices and accounting sync.

## Current phase

**Documentation first.** No application code until the PRDs and design docs for a phase are approved. Do not scaffold application code unless explicitly asked.

## Principles (see the ADRs)

- **Documentation first** (ADR 0011): no code without an approved PRD story and design doc. If the code proves a doc wrong, fix the doc in the same PR.
- **API first** (ADR 0010): one public API for the web app, customers and integrations. No private endpoints. Users are global (keyed by email) and gain access through memberships of accounts (business, customer, operations; operations implies no other access); each request acts as one membership. Actor-scoped record and field visibility in one central policy layer. OpenAPI contract written in the design doc first. Collection endpoints use RQL via Mpt.Rql with explicit per-property allow-lists; hidden fields can't be selected, filtered or sorted; limit max 1000, total always returned (ADR 0016).
- **Testing** (ADR 0012): every acceptance criterion has an automated test naming its story ID; every bug fix starts with a failing test; 95% line coverage gate on every code PR; real PostgreSQL via Testcontainers. Use Claude in Chrome for exploratory/smoke checks only when a change is visual or before a release.
- **Work in the open** (ADR 0013): be honest about bugs and mistakes, except security issues (private until fixed) and personal or shop data (never).
- **Build journal** (ADR 0014): every PR that changes the repo adds or updates `docs/journal/YYYY-MM-DD.md` (template in `docs/templates/`), including what the AI got wrong or was overruled on. Be strict; do not skip it. Add each new entry to the journal index and `mkdocs.yml` nav.
- **Multi-tenant** (ADR 0004): every tenant-owned table has `tenant_id`, enforced by PostgreSQL row-level security and EF Core query filters. Tenant context flows through jobs, storage, caches, logs and email. Every API test includes a cross-tenant case.
- **API conventions** (ADR 0020): paths `/api/v1/{module}/{collection}/{id}`, `PATCH` (JSON Merge Patch) not `PUT`, actions as `POST` verb sub-paths, no account in the path; app URLs mirror the API, one URL per object for every actor. Every request on an object (GET, POST, PATCH, action, DELETE) returns the full object, with hidden fields omitted.
- **Authentication** (ADR 0021): central OpenID Connect identity service at `id.signboard.works` (ASP.NET Core Identity + OpenIddict), single sign-on across business hosts; passkeys preferred, emailed link/code always available, passwords optional but always with a second factor; email links carry no credentials; integrations use hashed, scoped, expiring API tokens.
- **Object IDs** (ADR 0017): one ID per object, `PFX-NNNN-…` (random digits in groups of four, length fixed per type in its requirements, collision-checked by the primary key constraint with retry), used as the primary key everywhere. Versioned objects keep one ID. Register prefixes and lengths in `docs/design/data-model.md`.
- **Soft deletion** (ADR 0018): the default for every row; hard deletion only where a design doc justifies it.
- **Stack**: .NET LTS + ASP.NET Core, PostgreSQL via EF Core, React + TypeScript + Mantine SPA (ADRs 0007–0009); screens use our wrapper components, never Mantine directly; every list uses one `DataGrid` (TanStack Table, RQL server mode, per-user table views stored in the browser, business default views via the API). UI (static files on a CDN) and API (container) deploy separately but share one origin per business via an edge router routing `/api/*` (ADR 0015).

## Where things are

- `docs/product/`: vision and PRDs (user stories with Given/When/Then acceptance criteria)
- `docs/design/`: technical design documents
- `docs/adr/`: Architecture Decision Records (one decision each, numbered; edit freely during initial design, supersede once code implements them)
- `docs/templates/`: templates for new PRDs, design docs and ADRs
- `docs/glossary.md`: trade terms; add any new term used in a doc
- `docs/product/open-questions.md`: questions for the design partner shop

## Conventions

- The tenant is a **business** (business account), never a "shop" in the model, API or UI. "Shop" is fine in the trade sense ("sign shop", "shop floor", "the design partner shop").
- British English spelling throughout.
- Use standard industry terms: PRD, design document, ADR, user story, acceptance criteria. Do not use SoftwareOne-internal acronyms (BDR, TDR).
- User story IDs use a module prefix: QUO, SO, JOB, INV, EML, MIG.
- Every doc has a status header: Draft, In review, Approved, Superseded.
- Diagrams in Mermaid, not images.
- Never put the design partner shop's name, staff names, pricing or customers in the repo. Refer to it as "the design partner shop".
- Shop-specific behaviour is configuration, never code (ADR 0006).
- Code examples always use curly braces, even for single-line blocks.

## Commands

```bash
pip install -r requirements-docs.txt
mkdocs serve          # preview at http://127.0.0.1:8000
mkdocs build --strict # check for broken links
```

## Git

Commit or push only when Stuart asks.
