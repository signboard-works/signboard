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
- A decision is changed by writing a new ADR that supersedes the old one, never by editing history.
- Small overhead per decision, which is accepted.
