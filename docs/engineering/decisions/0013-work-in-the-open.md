# 0013. Work in the open

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard asks shops to trust it with their quotes, jobs and invoices. Shops leaving their current system are often doing so because of problems the vendor did not acknowledge or fix. Trust comes from seeing how a project handles its mistakes, not from claims that it has none.

## Decision

Nothing about how Signboard is built is secret.

- Plans, PRDs, designs, decisions and their reasoning are public in the repository.
- Bugs and known issues are tracked in public GitHub issues and described honestly, including their impact.
- Any incident that affects a shop's data or its customers gets a public post-incident review: what happened, why, and what changed.
- The changelog records fixes as plainly as features.
- How the project uses AI is documented openly, including where it went wrong.

Two limits apply, and both protect people rather than the project:

1. **Security vulnerabilities** are reported privately (GitHub private vulnerability reporting) and disclosed publicly in an advisory once a fix is released. How reports are handled, and the target times for fixes, are in [`SECURITY.md`](https://github.com/signboard-works/signboard/blob/main/SECURITY.md) and [ADR 0039](0039-security-baseline.md#vulnerability-response).
2. **Personal and business data** is never published: no shop, customer or staff names, no customer pricing. The design partner shop stays anonymous until it chooses otherwise.

## Options considered

1. **Work in the open, with limits for security and privacy**: builds trust and gives contributors full context. **Chosen.**
2. **Open code, private planning**: common in open source; contributors and shops see the result but not the reasoning.

## Consequences

- A `SECURITY.md` describing private vulnerability reporting is needed before the first release.
- Mistakes are visible, which is the point.
- Anonymised examples are needed wherever real shop data would help explain something.
