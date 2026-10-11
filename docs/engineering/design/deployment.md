# Design: Deployment

| | |
|---|---|
| **Status** | Draft |
| **Author** | Stuart Meeks |
| **Last updated** | 2026-10-10 |
| **Phase** | 1 |

## Context

How the hosted, multi-tenant Signboard service is deployed, backed up, monitored and upgraded, and how a business can run the same application itself as a single tenant ([ADR 0004](../decisions/0004-multi-tenant.md)).

## Known constraints (to be expanded)

- Hosted service: zero-downtime upgrades, monitoring, point-in-time database recovery, and per-tenant export and restore.
- Self-hosted: simple enough for a non-specialist to run, using Docker Compose with the same layout as the hosted service.
- Choose an OTLP-capable observability backend for logs, traces and metrics, with 90-day retention and email alerts ([ADR 0025](../decisions/0025-observability.md)).
- UI as static files behind a CDN, API as a container, one origin per business via an edge router ([ADR 0015](../decisions/0015-separate-deployments-one-origin.md)).
- Automated backups are part of the default install, not an extra.

## Design

_To do._

## Failure modes

_To do._

## Alternatives considered

_To do._

## Open questions

_To do._
