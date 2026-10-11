# 0003. Open-source licence

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard will be open source so any sign or print shop can use it. The licence must be chosen before the first code is committed, because changing it later requires agreement from every contributor.

The main tension is between maximum adoption and protection against someone hosting Signboard as a paid service without contributing improvements back. Signboard is not a money-making exercise, so protection against commercial use is not a goal. The maintainer intends to offer Signboard as a hosted service, possibly for profit ([ADR 0004](0004-multi-tenant.md)), and the licence should not stand in its way.

## Decision

We will license Signboard under the **Apache License 2.0**. The full text is in [`LICENSE`](https://github.com/signboard-works/signboard/blob/main/LICENSE) at the repository root.

## Options considered

1. **MIT**: maximally permissive and simple. Anyone can use, modify and resell it, including as a closed SaaS. No explicit patent grant.
2. **Apache 2.0**: permissive like MIT, with an explicit patent grant and patent-retaliation clause, and a clear contribution clause (section 5) so contributions are licensed on the same terms without a separate agreement. Widely accepted by companies. **Chosen.**
3. **AGPL-3.0**: anyone who runs a modified version as a network service must publish their changes. Protects against closed SaaS forks, but some companies avoid AGPL software, and it would bind any hosted offering by the maintainer to the same terms once outside contributions are merged.
4. **GPL-3.0**: copyleft for distributed software, but does not cover network use, so a hosted fork could stay closed. Gives little protection for a web application.

## Consequences

- Anyone, including the maintainer, may run Signboard as a paid hosted service, with or without publishing changes.
- Contributions are accepted under Apache 2.0 by default (section 5). No contributor licence agreement is needed.
- Dependencies must have licences compatible with Apache 2.0. GPL and AGPL libraries cannot be bundled. The full licence policy, enforced in CI, is in [ADR 0039](0039-security-baseline.md#supply-chain).
- Source files do not need a per-file licence header. The root `LICENSE` file is enough.
