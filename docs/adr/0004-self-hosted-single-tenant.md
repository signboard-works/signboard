# 0004. Self-hosted, single tenant

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard is built first for one shop and then offered to any shop that wants it. A multi-tenant SaaS would need tenant isolation, billing, support and operations that a part-time project cannot sustain.

## Decision

We will build Signboard as a self-hosted application where each installation serves exactly one shop. There is no tenant concept in the data model.

## Options considered

1. **Self-hosted, single tenant**: simplest data model and security model; each shop owns its data and its instance.
2. **Multi-tenant SaaS**: easier for non-technical shops to adopt; much higher build and operating cost, and a service someone must run.
3. **Single tenant now, tenant-ready schema**: keeps a door open, at the cost of complexity that may never be used.

## Consequences

- Deployment must be simple, with backups and upgrades built in.
- A hosted offering, if ever wanted, would mean running one instance per shop rather than changing the data model.
