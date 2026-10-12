# 0030. Hosting: AWS in Sydney

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

## Context

The hosted service needs somewhere to run. Earlier decisions have already set most of the shape:

- The UI is static files on a CDN, the API is a container, and an edge router serves both from one origin per business, sending `/api/*` to the API ([ADR 0015](0015-separate-deployments-one-origin.md)).
- There are four deployables: the UI, the API, the background worker and the identity service ([ADR 0026](0026-modular-monolith.md)), on hosts `<business>.signboard.works`, `signboard.works`, `id.signboard.works` and `ops.signboard.works` ([ADR 0020](0020-url-structure.md)).
- The database is PostgreSQL with row-level security ([ADR 0004](0004-multi-tenant.md), [ADR 0008](0008-database-postgresql.md)).
- Files go to S3-compatible storage with signed URLs, cheaper tiers for old files, and malware scanning ([ADR 0027](0027-file-storage.md)).
- Email is sent with Amazon SES in Sydney ([ADR 0023](0023-email-sending.md)).
- Logs, traces and metrics are exported with OTLP to a backend chosen with hosting ([ADR 0025](0025-observability.md)).
- Each pull request gets a preview environment, and `main` deploys to staging ([ADR 0028](0028-branching-ci-and-releases.md)).

The customers are Australian businesses, whose data is covered by the Privacy Act. The maintainer works part-time, so the platform must need little looking after. Self-hosters run the same application with Docker Compose, so nothing in the product may depend on one provider.

## Decision

We will host Signboard on **Amazon Web Services in the Sydney region (`ap-southeast-2`)**.

### Where data lives

Every service that stores data runs in Sydney: the database, its backups, files, email, secrets, logs, traces and metrics. The one exception is the backup copies kept outside production, which are in AWS Melbourne, also in Australia ([ADR 0043](0043-backups-and-recovery.md)). CloudFront serves the UI from edge locations, but caches only the static UI, never API responses. Customer and business data stays in Australia.

### Accounts and access

- An **AWS Organization** with four accounts: a management account for billing only, **production**, **non-production** (staging and preview environments) and, from before the first paying business, **backup**, which holds locked copies of production's backups in Melbourne ([ADR 0043](0043-backups-and-recovery.md)). The account boundaries keep a mistake in staging from touching production, and a compromise of production from deleting its backups.
- People sign in through **IAM Identity Center** with multi-factor authentication. Nobody uses the root user or long-lived access keys.
- **GitHub Actions deploys through OpenID Connect**, assuming a role in the target account. There are no AWS keys stored in GitHub.

### Services

| Need | Service |
|---|---|
| Edge router, CDN and TLS ([ADR 0015](0015-separate-deployments-one-origin.md)) | **CloudFront**, on a flat-rate plan (Pro for production, Free for non-production), with its included firewall and DNS. The default behaviour serves the UI from S3; `/api/*` goes to the API, and `/ingest/*` to PostHog ([ADR 0024](0024-product-analytics.md)). The identity service has its own distribution |
| Containers: API, worker, identity service | **ECS on Fargate**, ARM (Graviton), one service per deployable, rolling deploys that start new tasks before stopping old ones |
| Routing to containers | An **internal Application Load Balancer**, reachable only from CloudFront through a VPC origin |
| Database | **RDS for PostgreSQL**, encrypted, with automated backups and point-in-time recovery |
| Files and UI assets | **S3**, with all public access blocked. CloudFront reads UI assets through origin access control. Files use signed URLs and lifecycle rules for cheaper tiers ([ADR 0027](0027-file-storage.md)) |
| Malware scanning | **GuardDuty Malware Protection for S3** scans each upload. The worker records the result on the file ([ADR 0027](0027-file-storage.md)) |
| Email | **SES** in Sydney, with delivery events through SNS ([ADR 0023](0023-email-sending.md)) |
| Notifications to the worker | **SQS**: SES delivery events and GuardDuty scan results arrive on a queue the worker reads ([ADR 0034](0034-background-work.md)) |
| Secrets and configuration | **Secrets Manager** for credentials, with rotation; **Parameter Store** for other settings |
| Container images | **ECR** for the hosted service. Releases are also published to GitHub Container Registry for self-hosters ([ADR 0028](0028-branching-ci-and-releases.md)) |
| DNS | **Route 53** hosts the `signboard.works` zone. `docs.signboard.works` stays a CNAME to GitHub Pages |
| Observability backend ([ADR 0025](0025-observability.md)) | An **AWS Distro for OpenTelemetry** collector sends logs to **CloudWatch Logs**, traces to **X-Ray** and metrics to **CloudWatch**. Log groups keep 90 days. Alerts are CloudWatch alarms that email through SNS |
| Cost control | **AWS Budgets** alerts at A$350 and A$500 a month |

### Network

- One VPC per account, across two availability zones.
- The database sits in private subnets with no route to the internet.
- Fargate tasks sit in public subnets with public IPv4 addresses, so they can reach AWS services and the internet without a NAT gateway. Their security groups accept traffic only from the load balancer. A NAT gateway would cost about US$45 a month per environment for no benefit at this size.

### Resilience

- **At launch:** the database runs in one availability zone, with automated backups kept for 14 days and point-in-time recovery. Each service runs one task, and ECS replaces a failed task within a minute or two.
- **Before the first paying business:** the database moves to **Multi-AZ**, with a standby in a second zone and automatic failover. The API and identity service run at least two tasks, in different zones.
- Recovery targets, backup copies outside production and restore testing are decided in [ADR 0043](0043-backups-and-recovery.md).

### Environments

| Environment | Account | Shape |
|---|---|---|
| Production | Production | As above |
| Staging | Non-production | The same shape at the smallest sizes, a `db.t4g.micro` database, and services stopped at nights and weekends. `main` deploys here automatically |
| Preview (one per pull request) | Non-production | ECS services for the pull request, sharing staging's load balancer and database server, with a database of its own. Created when the pull request opens, removed when it closes, stopped at night |

Host names for staging and previews are set in the deployment design.

### Infrastructure as code

- All infrastructure is defined with the **AWS CDK in C#**, in the repository, and deployed by GitHub Actions. The same code builds every environment, with sizes as configuration.
- Nobody changes infrastructure by hand in the console. A manual change in an emergency is copied into the code straight away.

### Expected cost

Prices from AWS's price list for Sydney on 2026-10-11, using ARM processors, converted at A$1.43 to the US dollar and including 10% GST. A month:

| | US$ | A$ incl. GST |
|---|---:|---:|
| Production at launch | 130 | 204 |
| Production before paying businesses (Multi-AZ database, two tasks per web service) | 211 | 332 |
| Staging | 58 | 91 |
| Preview environments (about 30 pull-request days) | 18 | 28 |
| **Total at launch** | **206** | **324** |
| **Total before paying businesses** | **287** | **452** |

The largest items are the database (about 30–35% of production), the load balancer and Fargate. Not included: growth beyond the first handful of businesses, traces and metrics beyond basic volumes, and the domain registration. A one-year commitment (reserved database instances and a Compute Savings Plan) would cut roughly 20–35% from the database and Fargate lines once usage is steady. Credits from AWS programmes for new accounts and open-source projects should be applied for. The security services in [ADR 0039](0039-security-baseline.md) (GuardDuty threat detection and KMS keys) add about A$15 a month, bringing the launch total to about A$340. Backup copies to Melbourne and restore tests ([ADR 0043](0043-backups-and-recovery.md)) add about A$10 a month before paying businesses, bringing that total to about A$475. PDF rendering ([ADR 0048](0048-documents-and-templates.md#where-rendering-runs)) adds about A$5 a month at launch, for more worker memory, and about A$25 before paying businesses, for a renderer task of its own: about A$345 and A$500.

### Staying portable

Self-hosting must keep working without AWS ([ADR 0015](0015-separate-deployments-one-origin.md)). Every AWS service used is behind a standard interface or a small adapter:

| Hosted on AWS | Self-hosted equivalent |
|---|---|
| ECS on Fargate | Docker Compose |
| RDS for PostgreSQL | PostgreSQL container |
| S3 | Any S3-compatible store, such as MinIO |
| GuardDuty malware scanning | A ClamAV container, behind the same scanning interface |
| SES | SMTP ([ADR 0023](0023-email-sending.md)) |
| CloudFront and the load balancer | Caddy ([ADR 0015](0015-separate-deployments-one-origin.md)) |
| CloudWatch and X-Ray | Any OTLP backend, or console logs ([ADR 0025](0025-observability.md)) |
| Secrets Manager | Environment variables or Docker secrets |

## Options considered

1. **AWS in Sydney**: SES is already chosen; S3 is the reference for S3-compatible storage; CloudFront matches the edge router design exactly; GuardDuty scans uploads without a scanner to run; every service runs in Sydney; CDK can be written in C#. More services and settings to learn, and no scale-to-zero for containers. **Chosen.**
2. **Azure in Australia East**: Container Apps is simpler than ECS and scales to zero, which suits preview environments; strong .NET support and the maintainer's familiarity with Microsoft. Blob Storage is not S3-compatible, which would change ADR 0027; email would stay on SES, so two clouds from day one; Front Door's base price is higher than CloudFront's.
3. **Google Cloud in Sydney**: Cloud Run scales to zero and Cloud SQL is solid. No advantage over AWS for our fit, and email and storage would still point at AWS.
4. **DigitalOcean in Sydney**: cheaper and simpler, with S3-compatible Spaces and managed PostgreSQL. No managed malware scanning, fewer managed services, and less room to grow.
5. **Fly.io, Render or Railway**: very simple to deploy. Weak or no Australian presence for managed PostgreSQL, so data residency and latency suffer.
6. **Virtual servers (for example Hetzner)**: the cheapest. Every operational task falls on the maintainer, and there is no Australian region.

## Consequences

- The deployment design doc fills in the CDK stacks, CloudFront distributions and behaviours, host names for staging and previews, health check paths and the deploy pipeline.
- [ADR 0025](0025-observability.md)'s backend is now CloudWatch and X-Ray. Because the code only speaks OTLP, moving to another backend later is a collector configuration change.
- Malware scanning on AWS happens in GuardDuty rather than in the worker. The worker still owns the file's scan status, and self-hosted installations use the same interface with ClamAV.
- The `signboard.works` DNS zone moves to Route 53, with SES domain verification (DKIM) records created by the CDK code.
- Running costs start at about A$324 a month and are reviewed against the budget alerts. Moving the database to Multi-AZ before the first paying business is a planned step, not an option.
- Custom domains for businesses, when they come, need a CloudFront set-up that supports many domains on one distribution. That is costed and designed then.
- The maintainer needs working knowledge of the AWS services above. Keeping the set small and defined in code limits that.
