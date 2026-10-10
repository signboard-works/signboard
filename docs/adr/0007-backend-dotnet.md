# 0007. Back end: .NET and ASP.NET Core

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Signboard is maintained in evenings and weekends, so the back end must be something the maintainer can debug quickly after a fortnight away ("boring technology" in the [vision](../product/vision.md)). The maintainer's deep experience is in .NET.

## Decision

We will build the back end in C# on the current .NET long-term support release (.NET 10 at the time of writing), using ASP.NET Core for the HTTP API ([ADR 0010](0010-api-first.md)) and Entity Framework Core for data access ([ADR 0008](0008-database-postgresql.md)). We will move to each new LTS release within six months of its release.

## Options considered

1. **.NET (ASP.NET Core)**: the maintainer's strongest skill; fast, mature, cross-platform, runs well in a small Linux container; first-class OpenAPI support. **Chosen.**
2. **Node.js (TypeScript)**: one language across front and back end; less familiar to the maintainer, and a weaker standard library for domain-heavy code such as pricing.
3. **Python (Django)**: excellent admin and ORM; unfamiliar to the maintainer.

## Consequences

- The front end and back end are in different languages (TypeScript and C#). The OpenAPI contract is the bridge, and the TypeScript client is generated from it.
- Contributors need .NET skills for back-end work.
- Only LTS releases are used, so upgrades happen roughly every two years rather than every year.
