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
- Hosted on AWS in Sydney: CloudFront, ECS on Fargate behind an internal load balancer, RDS for PostgreSQL, S3, GuardDuty malware scanning, SES, Route 53, defined in AWS CDK (C#) and deployed from GitHub Actions through OpenID Connect ([ADR 0030](../decisions/0030-hosting-aws.md)).
- Three AWS accounts: management, production, non-production (staging and preview environments).
- Observability: an AWS Distro for OpenTelemetry collector to CloudWatch Logs, X-Ray and CloudWatch metrics, 90-day retention, alarms by email ([ADR 0025](../decisions/0025-observability.md)).
- To design here: CDK stacks, CloudFront distributions and behaviours, host names for staging and previews, health check paths, the deploy pipeline.
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
