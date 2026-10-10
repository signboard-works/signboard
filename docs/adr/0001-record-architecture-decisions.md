# 0001. Record architecture decisions

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard is open source and maintained part-time. Decisions made early will be questioned later by contributors, and by the maintainers themselves after time away. Without a record, the reasoning is lost and decisions get relitigated or quietly reversed.

## Decision

We will record every architecturally significant decision as an Architecture Decision Record in `docs/adr/`, numbered sequentially, using the format described by Michael Nygard.

## Consequences

- Reasoning survives beyond the people who made the decision.
- During initial design, before any code implements it, an accepted ADR may be edited directly; Git keeps the history of how it changed. Once code implements an ADR, it is changed only by a new ADR that supersedes it.
- Small overhead per decision, which is accepted.
