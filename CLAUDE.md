# Signboard: working notes for Claude

Signboard is open-source, multi-tenant job management for sign and print shops: quotes, sales orders, jobs, a production kanban, invoices and accounting sync.

## Current phase

**Documentation first.** No application code until the PRDs and design docs for a phase are approved. Do not scaffold application code unless explicitly asked.

## Principles (see the ADRs)

- **Documentation first** (ADR 0011): no code without an approved PRD story and design doc. If the code proves a doc wrong, fix the doc in the same PR.
- **API first** (ADR 0010): one public API for the web app, customers and integrations. No private endpoints. Users are global (keyed by email) and gain access through memberships of accounts (business, customer, operations; operations implies no other access); each request acts as one membership. Actor-scoped record and field visibility in one central policy layer. OpenAPI contract written in the design doc first. Collection endpoints use RQL via Mpt.Rql with explicit per-property allow-lists; hidden fields can't be selected, filtered or sorted; limit max 1000, total always returned (ADR 0016).
- **Testing** (ADR 0012): every acceptance criterion has an automated test naming its story ID; every bug fix starts with a failing test; 95% line coverage gate on every code PR; real PostgreSQL via Testcontainers. Use Claude in Chrome for exploratory/smoke checks only when a change is visual or before a release.
- **Work in the open** (ADR 0013): be honest about bugs and mistakes, except security issues (private until fixed) and personal or shop data (never).
- **Build journal** (ADR 0014): every PR that changes the repo adds or updates `docs/journal/YYYY-MM-DD.md` (template in `docs/engineering/templates/`), including what the AI got wrong or was overruled on. Be strict; do not skip it. Add each new entry to the journal index and the `zensical.toml` nav.
- **Branching, CI and releases** (ADR 0028): trunk-based; short-lived `type/description` branches, squash-merged; PR titles follow Conventional Commits (`feat(sales): …`, `docs: …`) because they become the commit on `main`; required checks on every PR, Playwright `@smoke` tier on PRs and full suite on `main`/nightly/release; preview environment per PR; semver releases via release-please; Dependabot; third-party actions pinned by SHA; never `pull_request_target`.
- **Multi-tenant** (ADR 0004): every tenant-owned table has `tenant_id`, enforced by PostgreSQL row-level security and EF Core query filters. Tenant context flows through jobs, storage, caches, logs and email. Every API test includes a cross-tenant case.
- **API conventions** (ADR 0020): paths `/api/v1/{module}/{collection}/{id}`, `PATCH` (JSON Merge Patch) not `PUT`, actions as `POST` verb sub-paths, no account in the path; app URLs mirror the API, one URL per object for every actor. Every request on an object (GET, POST, PATCH, action, DELETE) returns the full object, with hidden fields omitted.
- **API errors, idempotency and versioning** (ADR 0029): every error is RFC 9457 problem details with our own `type` page, a stable snake_case `code`, the bare `traceId`, and `errors` as `{pointer, code, message}`; 400 unreadable, 422 invalid values, 403 `forbidden` or `not_entitled`, 404 for anything the caller can't see; unknown or unwritable fields refused; optional `Idempotency-Key` on POST (24 h, sent by the generated client); additive changes stay in `v1`, breaking changes need `v2` with a 12-month overlap (free change before public launch); expand-then-contract so the API works with the previous UI build.
- **Authentication** (ADR 0021): central OpenID Connect identity service at `id.signboard.works` (ASP.NET Core Identity + OpenIddict), single sign-on across business hosts; passkeys preferred, emailed link/code always available, passwords optional but always with a second factor; email links carry no credentials; integrations use hashed, scoped, expiring API tokens.
- **Authorisation** (ADR 0022): code-defined permissions (`module.collection.action`), deny by default, evaluated per request; fields belong to field groups; built-in roles only (several per membership); ops self-add gets read-only Signboard support role.
- **Email** (ADR 0023): separate business and platform streams; business's own verified domain, falling back to "via Signboard"; outbox plus delivery events on every email; suppression lists; no open/click tracking; SES in Sydney, SMTP for self-hosters.
- **Product analytics** (ADR 0024): PostHog Cloud (EU), named events only, users identified by object ID, no business data in events, session replay off, proxied via `/ingest`; self-hosted sends nothing by default. Each feature design lists its events.
- **Concurrency** (ADR 0031): optimistic; every aggregate root has `revision` (incremented by a PostgreSQL `BEFORE UPDATE` trigger, read back with `RETURNING`; owned-row changes bump the parent) and `updated_at`; `ETag: "<revision>"` plus a read-only `revision` field; `If-Match` required on PATCH, DELETE and actions (428 missing, 412 stale with `currentRevision`); `If-Match: *` allowed for integrations; `If-None-Match` gives 304; UI reapplies non-overlapping edits, otherwise shows a conflict dialog.
- **Events, webhooks and live updates** (ADR 0032): one transactional event outbox in PostgreSQL; change events for every object come from the revision trigger, named business events (`{module}.{object}.{verb}`) from code; at-least-once, consumers idempotent; 7-day retention. Live updates for the whole UI over Server-Sent Events (pending a CloudFront spike, SignalR the fallback): thin notifications only, filtered by visibility, client refetches through the API. Webhooks follow Standard Webhooks, business accounts only at launch, SSRF protections.
- **Feature flags** (ADR 0033): release flags (temporary) and `ops.*` kill switches only, never entitlements, settings or permissions; declared in code with owner and remove-by date (90 days), CI fails when overdue, retiring deletes flag and old path together; state in PostgreSQL via Microsoft.FeatureManagement behind our own `IFeatures`; targeting by environment, business, user, percentage of businesses; only operations users switch flags; off means the feature doesn't exist (404, fields omitted, not in the public reference); previews all on, production off.
- **Background work** (ADR 0034): our own job queue in PostgreSQL (`SKIP LOCKED`, `LISTEN/NOTIFY`, Cronos for schedules), shared with the event outbox; no Redis or broker. Jobs are enqueued in the same transaction as the change and carry business, actor and trace ID (worker sets tenant context, so RLS applies); at-least-once, handlers idempotent; backoff retries, then a failed state with retry/discard in the ops app and an alert; per-business fairness, per-key ordering, per-type limits; business-facing schedules in the business's time zone; AWS notifications arrive via SQS.
- **Retries and resilience** (ADR 0035): API requests never wait on outside services (that work is a job); two layers only (short retries around a call, then job retries), never stacked; every outside call has a timeout; retry only temporary failures (network, timeouts, 408, 429, 502–504) and only safe-to-repeat operations, honour `Retry-After`, exponential backoff with jitter; circuit breaker and rate limiter per destination; named pipelines defined centrally, never hand-written loops. Microsoft.Extensions.Http.Resilience (Polly) for HTTP; AWS SDK and EF Core's execution strategy use their own retries; fault-injection tests with Polly chaos and WireMock.Net.
- **Hosting** (ADR 0030): AWS Sydney (`ap-southeast-2`), all data in Australia; CloudFront (flat-rate) as edge router, ECS Fargate (ARM) behind an internal ALB via VPC origin, RDS PostgreSQL (single-AZ at launch, Multi-AZ before paying businesses), S3, GuardDuty malware scanning, SES, Route 53; accounts: management, production, non-production; GitHub Actions deploys via OIDC; all infrastructure in AWS CDK (C#), no console changes; every AWS service has a self-hosted equivalent.
- **Observability** (ADR 0025): OpenTelemetry logs, traces and metrics via `ILogger` and OTLP; structured fields with IDs only, never business or personal data (redaction enforced); trace ID = audit correlation ID; front-end errors to PostHog; 90-day retention; email alerts, nothing out of hours. Backend: ADOT collector to CloudWatch Logs, X-Ray and CloudWatch metrics (ADR 0030).
- **Modular monolith** (ADR 0026): one codebase, one database; each module is its own project owning its own tables; never touch another module's tables, call its interface; architecture tests enforce it. Deployables: UI, API, worker, identity service.
- **File storage** (ADR 0027): one files component in `system`; files are `FIL-` objects with metadata in PostgreSQL and content in S3-compatible storage keyed `{tenantId}/{fileId}`; signed direct upload/download; 50 MB limit (configurable); access follows the attached object plus audience (internal or shared); immutable, scanned, sent PDFs kept exactly.
- **Object IDs** (ADR 0017): one ID per object, `PFX-NNNN-…` (random digits in groups of four, length fixed per type in its requirements, collision-checked by the primary key constraint with retry), used as the primary key everywhere. Versioned objects keep one ID. Register prefixes and lengths in `docs/engineering/design/data-model.md`.
- **Soft deletion** (ADR 0018): the default for every row; hard deletion only where a design doc justifies it.
- **Stack**: .NET LTS + ASP.NET Core, PostgreSQL via EF Core, React + TypeScript + Mantine SPA (ADRs 0007–0009); screens use our wrapper components, never Mantine directly; every list uses one `DataGrid` (TanStack Table, RQL server mode, per-user table views stored in the browser, business default views via the API). UI (static files on a CDN) and API (container) deploy separately but share one origin per business via an edge router routing `/api/*` (ADR 0015).

## Where things are

The site has four tabs, each a top-level section of `nav` in `zensical.toml`: Overview, Engineering, API reference and Journal. Keep the left nav short: new pages go inside an existing section, and new ADRs go in the matching theme group.

- `docs/index.md`, `docs/vision.md`, `docs/glossary.md`: Overview tab. Add any new trade term used in a doc to the glossary.
- `docs/engineering/requirements/`: PRDs (user stories with Given/When/Then acceptance criteria) and `open-questions.md` (questions for the design partner shop)
- `docs/engineering/design/`: technical design documents
- `docs/engineering/decisions/`: Architecture Decision Records (one decision each, numbered; edit freely during initial design, supersede once code implements them), grouped by theme in the nav
- `docs/engineering/templates/`: templates for new PRDs, design docs, ADRs and journal entries (not published)
- `docs/api/`: API reference: shared conventions now, then one page per module generated from the OpenAPI contract
- `docs/journal/`: build journal

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
zensical serve          # preview at http://localhost:8000
zensical build --strict # check for broken links
```

## Git

Commit or push only when Stuart asks.
