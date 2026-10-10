# 0003. Open-source licence

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard will be open source so any sign or print shop can use it. The licence must be chosen before the first code is committed, because changing it later requires agreement from every contributor.

The main tension is between maximum adoption and protection against someone hosting Signboard as a paid service without contributing improvements back.

## Decision

_Not yet decided._

## Options considered

1. **MIT**: maximally permissive and simple. Anyone can use, modify and resell it, including as a closed SaaS.
2. **Apache 2.0**: permissive like MIT, with an explicit patent grant. Common for larger projects.
3. **AGPL-3.0**: anyone who runs a modified version as a network service must publish their changes. Protects against closed SaaS forks; some companies avoid AGPL software.
4. **GPL-3.0**: copyleft for distributed software, but does not cover network use, so a hosted fork could stay closed.

## Consequences

To be written once decided.
