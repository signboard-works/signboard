# 0012. Testing strategy

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard handles a shop's quotes, prices and invoices. A wrong price or a lost job costs the shop real money and trust. The maintainer works in short sessions, often with an AI assistant writing code, so automated tests are the main protection against regressions. Every PRD already states its acceptance criteria as Given/When/Then.

## Decision

Tests are written as part of every change, at the level that gives the most confidence for the least cost:

| Level | Tools | Covers |
|---|---|---|
| Unit | xUnit | Domain logic, especially the pricing engine and state machines |
| Integration | xUnit, ASP.NET Core `WebApplicationFactory`, Testcontainers (PostgreSQL) | Every API endpoint through the real HTTP pipeline and a real database, called as each actor type, including field visibility ([ADR 0010](0010-api-first.md)) and cross-tenant isolation ([ADR 0004](0004-multi-tenant.md)) |
| Contract | OpenAPI validation in the integration tests | Responses match the published contract |
| Front-end component | Vitest, React Testing Library, Mock Service Worker | Component behaviour against mocked API responses |
| End to end | Playwright | A small set of critical journeys, such as quote to sales order to job to invoice |
| Exploratory and smoke | Claude in Chrome | Visual and usability checks on UI changes and before releases |

Rules:

- Every acceptance criterion has at least one automated test that names its story ID (for example `QUO-003`), so coverage of the requirements can be traced.
- Every bug fix starts with a failing test that reproduces the bug.
- The pricing engine is also tested against real past jobs from the design partner shop (anonymised), and with property-based tests for rounding and tax.
- Every endpoint has a cross-tenant test, and a schema test fails if any tenant-owned table lacks `tenant_id` or a row-level security policy.
- Tests use real PostgreSQL, never an in-memory database substitute.
- All unit, integration, contract and component tests run in CI on every pull request and must pass before merge. End-to-end tests run in tiers: a smoke subset on every pull request and the full suite on `main`, nightly and before every release ([ADR 0028](0028-branching-ci-and-releases.md)).
- Every pull request that changes code must keep line coverage at or above **95%**, measured separately for the back end (coverlet) and the front end (Vitest). CI fails the pull request below that threshold. The gate applies from the first line of code, so it never has to be retrofitted. Generated code (EF Core migrations, the generated API client) is excluded, and every other exclusion must be explicit in configuration with a comment giving the reason.
- Coverage is a floor, not the goal. Reviews check that tests assert behaviour, and acceptance-criterion traceability still applies.
- Exploratory testing with Claude in Chrome is used when a change is visual or interaction-heavy, or before a release. It is not a merge gate, and its findings become issues or automated tests.

## Options considered

1. **Test pyramid plus traceable acceptance criteria and AI-driven exploratory checks**: strong regression protection, requirements visibly covered, and manual checking kept proportionate. **Chosen.**
2. **Mostly end-to-end tests**: close to real use; slow, brittle and hard to diagnose.
3. **Unit tests only with a coverage target**: fast; misses integration, authorisation and database behaviour.
4. **No coverage gate, acceptance-criterion traceability only**: avoids tests written to hit a number; lets untested code creep in during short, AI-assisted sessions.

## Consequences

- Docker is required for development and in CI, for Testcontainers.
- Acceptance criteria must be written precisely enough to test, which raises the bar for PRDs.
- Playwright journeys need maintaining as the UI changes, so they are kept few.
- A 95% gate from the start means tests are written with the code, never afterwards. It also risks tests written to hit the number, which reviews must watch for.
