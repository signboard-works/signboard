# 0014. Build journal

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard is built with heavy AI assistance, and the project intends to publish an honest account of how ([ADR 0011](0011-documentation-first.md), [ADR 0013](0013-work-in-the-open.md)). Git history and the docs record *what* changed. They do not record why a session went the way it did, what the AI got wrong, or what was corrected. That context is lost within days if it is not written down, and a journal kept "when there's time" stops being kept.

## Decision

We will keep a dated build journal in `docs/journal/`, published with the docs site.

- Every working session that changes the repository adds or updates a journal entry, **in the same pull request** as the change.
- One file per day worked, named `YYYY-MM-DD.md`, using the [journal template](https://github.com/signboard-works/signboard/blob/main/docs/engineering/templates/journal-template.md).
- Each entry records what was done, the decisions made and by whom, and where the AI was wrong or was overruled. What comes next is on the project board ([ADR 0040](0040-backlog-and-issues.md)).
- Entries are honest and specific. Mistakes are recorded, not tidied away.
- Entries follow the same privacy limits as everything else: no shop, customer or staff names, and no customer pricing.
- A CI check fails any pull request that changes files outside `docs/journal/` without also changing a file inside it. Trivial changes (typos, dependency bumps) may skip it with a `no-journal` label, used sparingly.

## Options considered

1. **Journal in the repo, updated in every pull request, enforced by CI**: hard to forget; the record is complete and versioned. **Chosen.**
2. **Journal in the repo, by convention only**: no tooling; in practice it lapses.
3. **Rely on pull request descriptions**: already written; scattered, not readable as a story, and not part of the docs site.

## Consequences

- Every pull request carries a small amount of extra writing.
- The journal becomes the raw material for the "how we built this with AI" site.
- The `no-journal` label needs watching, so it does not become the default.
