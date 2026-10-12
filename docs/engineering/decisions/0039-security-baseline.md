# 0039. Security baseline

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

## Context

Signboard holds businesses' prices, customers, files and accounting connections, and every business shares one hosted service ([ADR 0004](0004-multi-tenant.md)). A breach would expose many businesses at once, and the code is public, so attackers can read it as easily as contributors.

Much of Signboard's security is already decided:

| Area | Decided in |
|---|---|
| Tenant isolation by row-level security and query filters, checked twice | [ADR 0004](0004-multi-tenant.md) |
| Field visibility per actor, and queries that cannot reach hidden fields | [ADR 0010](0010-api-first.md), [ADR 0016](0016-rql-queries.md) |
| Private reporting of vulnerabilities | [ADR 0013](0013-work-in-the-open.md) |
| UI and API on one origin, so the API is the only attack surface | [ADR 0015](0015-separate-deployments-one-origin.md) |
| Audit trail, including security events | [ADR 0019](0019-audit-trail.md) |
| Sign-in, passkeys, second factors and API tokens | [ADR 0021](0021-authentication.md) |
| Permissions, deny by default | [ADR 0022](0022-authorisation-and-roles.md) |
| Logs and traces with IDs only, redaction enforced | [ADR 0025](0025-observability.md) |
| Malware scanning of uploaded files | [ADR 0027](0027-file-storage.md) |
| CodeQL, dependency review, secret scanning, pinned actions, no `pull_request_target` | [ADR 0028](0028-branching-ci-and-releases.md) |
| Webhook signing and protection against server-side request forgery | [ADR 0032](0032-events-webhooks-live-updates.md) |
| Rate limiting, and sign-in and abuse limits | [ADR 0038](0038-rate-limiting.md) |

This ADR sets the standard all of that is measured against, and fills the gaps: browser security, secrets and encryption, the software supply chain, testing, vulnerability response, and access to the hosted service.

## Decision

### The standard

- Signboard targets **OWASP ASVS 5.0 Level 2** (the Application Security Verification Standard) across the whole product. Level 2 is meant for applications that handle sensitive business data.
- **Each design doc has a security section.** Where a feature crosses a trust boundary, such as customer access, files, webhooks, integrations or anything a business can configure, the section includes a short threat model: what could go wrong, and what prevents it.

### Browser security

**Headers.** The static UI's headers are added by a CloudFront Function, because custom response header policies are not in the CloudFront Pro plan ([ADR 0030](0030-hosting-aws.md)). The API, identity service and every error page set their own.

| Header | Value |
|---|---|
| `Content-Security-Policy` | Scripts only from our own origin, no `eval` or inline scripts, `frame-ancestors 'none'`, `object-src 'none'`, `base-uri 'none'`, connections only to our own origin (PostHog is proxied through `/ingest`, [ADR 0024](0024-product-analytics.md)). If Mantine's inline styles need `style-src 'unsafe-inline'`, the front-end design says so; scripts never do |
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload`, with `signboard.works` submitted to the browsers' preload list |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Everything off that Signboard does not use, such as camera, microphone and geolocation |
| `Cross-Origin-Opener-Policy` | `same-origin` |

Content Security Policy violations are reported to an API endpoint and counted in metrics ([ADR 0025](0025-observability.md)).

**Cookies.**

- The session cookie is named with the `__Host-` prefix, and is `Secure`, `HttpOnly` and `SameSite=Lax`. The prefix ties it to the exact host and stops subdomains from setting or overwriting it.
- No other cookie holds anything sensitive, and browser code never sees a token ([ADR 0021](0021-authentication.md)).

**Cross-site request forgery.** Every request that changes something (`POST`, `PATCH`, `DELETE`) and is authenticated by the session cookie must:

- come from our own origin, according to the browser's Fetch Metadata header (`Sec-Fetch-Site: same-origin`);
- carry an `Origin` header matching the host;
- send a JSON body (`application/json`), which a plain HTML form cannot.

Requests authenticated with an API token carry no cookie, so they are not exposed to this attack and these checks do not apply to them. Every current browser sends these headers, so no anti-forgery token is needed.

**CORS.** The API sends no CORS headers. The web app is on the same origin ([ADR 0015](0015-separate-deployments-one-origin.md)) and integrations call from servers with tokens, so no other website's scripts can call the API. Browser-based third-party apps are not supported at launch.

### Secrets

- **Platform credentials**, such as the database password and the SES and PostHog keys, are in AWS Secrets Manager, with rotation where AWS supports it ([ADR 0030](0030-hosting-aws.md)). They are never in the repository, in images or in plain environment files. Self-hosted installations use Docker secrets.
- **Secrets that belong to a business**, such as accounting OAuth tokens ([ADR 0005](0005-accounting-as-plugin.md)) and webhook signing secrets ([ADR 0032](0032-events-webhooks-live-updates.md)), are encrypted by the application before they are stored, using **envelope encryption**: each is encrypted with a data key, and data keys are encrypted with one AWS KMS key per environment. A database dump alone reveals none of them. Self-hosted installations use a key from configuration in place of KMS.
- **ASP.NET Core Data Protection keys**, which protect session cookies and other short-lived tokens, are stored in PostgreSQL, encrypted the same way, so every task can read cookies issued by any other.
- Secrets are never logged, returned by the API after they are created, or included in error messages, events or webhooks.

### Encryption in transit

| Connection | TLS |
|---|---|
| Browsers and integrators to CloudFront | **TLS 1.3 only** (CloudFront's `TLSv1.3_2025` policy), which also gives CloudFront's quantum-safe key exchange |
| CloudFront to the load balancer | **TLS 1.2 minimum**, with strong ciphers only. CloudFront connects to origins over TLS 1.2 at most. The connection stays inside AWS, through the private VPC origin ([ADR 0030](0030-hosting-aws.md)) |
| Load balancer to API tasks | Inside the VPC, accepted only from the load balancer's security group |
| API and worker to PostgreSQL | **TLS 1.3 only**, required by the database |
| Calls to AWS services | The highest version both sides support, normally TLS 1.3 |
| Calls to other services, such as Xero and webhook receivers | **TLS 1.2 minimum**, because we cannot require other people's servers to support TLS 1.3 |

Self-hosted installations get the same defaults in the documented proxy configuration. If a real integrator cannot connect with TLS 1.3, allowing TLS 1.2 at CloudFront is a one-line change. Tightening it later would break people, so we start strict.

### Encryption at rest

- The database, its backups and snapshots, file storage, logs and the job queue are encrypted at rest with AWS-managed keys.
- Personal data is not separately encrypted column by column. It is protected by row-level security ([ADR 0004](0004-multi-tenant.md)), field visibility ([ADR 0010](0010-api-first.md)) and encryption of the database itself. Business secrets are the exception, as above.

### Credentials

- **Passwords**, where a user chooses one ([ADR 0021](0021-authentication.md)), are hashed with ASP.NET Core Identity's PBKDF2 hasher with its iteration count raised to OWASP's current recommendation, 600,000 for PBKDF2-HMAC-SHA256, and raised again when that recommendation changes. Hashes are upgraded when users next sign in.
- **API tokens and emailed codes** are high-entropy random values, so they are stored as SHA-256 hashes, which are fast to check without weakening them. Each token has a recognisable prefix, such as `sbt_`, so secret scanning can spot one that has leaked.

### Safe code

- **No string-built SQL.** Database access goes through EF Core or parameterised queries. Raw SQL is allowed only with parameters, and is reviewed.
- **No HTML from users is rendered as HTML** in the app. Where a feature needs formatted text written by a business, such as templates, it is sanitised with HtmlSanitizer (MIT) and shown in a sandbox. The rules for templates are decided with documents and templates.
- **Uploads** are size-limited, scanned and served with their own content type and `Content-Disposition` ([ADR 0027](0027-file-storage.md)).
- **Outgoing requests to addresses a user chose**, such as webhooks, are protected against server-side request forgery ([ADR 0032](0032-events-webhooks-live-updates.md)). Any new feature that fetches a URL uses the same protected client.

### Supply chain

**Dependency licences**, extending [ADR 0003](0003-licence.md):

| | Licences |
|---|---|
| **Allowed** | MIT, Apache 2.0, BSD-2-Clause, BSD-3-Clause, ISC, the PostgreSQL licence, Unicode, zlib |
| **Case by case**, with a reason in the pull request | MPL 2.0, LGPL |
| **Never** | GPL, AGPL, SSPL, BUSL, the Commons Clause and other source-available or non-commercial licences |

- CI enforces the list with GitHub's dependency review action ([ADR 0028](0028-branching-ci-and-releases.md)).
- **A new dependency needs a reason.** Its pull request says why it is needed, and that it is maintained and widely used. A few lines of our own code are preferred to a small, little-used package.
- Lock files are committed, and CI installs exactly what they list.

**Images and releases.**

- Container images use Microsoft's **chiseled Ubuntu images** for .NET, which have no shell or package manager and run as a non-root user. We use the `-extra` variant, which includes ICU ([ADR 0037](0037-localisation.md)). The one exception is the PDF renderer, which needs Chromium and its libraries ([ADR 0048](0048-documents-and-templates.md#where-rendering-runs)): it still runs as a non-root user with a read-only root filesystem, and has no database access, secrets or outbound network.
- Containers run with a read-only root filesystem.
- Every release publishes a **software bill of materials** (CycloneDX) and **signed build provenance** through GitHub's artifact attestations, so self-hosters can check that an image was built from this repository by our release workflow ([ADR 0028](0028-branching-ci-and-releases.md)).
- ECR scans images when they are pushed, and Dependabot keeps base images current.

### Finding problems

- **Deny-by-default tests.** Every endpoint has a test that a caller without the permission is refused, alongside its cross-tenant test ([ADR 0012](0012-testing.md)). An architecture test fails if an endpoint has no permission declared.
- **Scanning.** CodeQL and dependency review on every pull request ([ADR 0028](0028-branching-ci-and-releases.md)), and an **OWASP ZAP baseline scan** (Apache 2.0) every night against staging.
- **An independent penetration test of the whole product before the first paying business**, extending the identity service review already planned in [ADR 0021](0021-authentication.md). Then again after major changes, and at least yearly once there are paying businesses.

### Vulnerability response

- The repository has a **`SECURITY.md`** explaining how to report a vulnerability privately, through GitHub's private vulnerability reporting ([ADR 0013](0013-work-in-the-open.md)), and what reporters can expect.
- **Targets:** acknowledged within 3 days; critical issues fixed within 7 days, high within 30 days, others in a normal release.
- Fixed vulnerabilities are published as GitHub security advisories, with a CVE where it applies, once the fix is released.
- **Supported versions:** the latest release. Self-hosters get security fixes by upgrading.
- Dependabot security updates are on, and a security update to a dependency is merged ahead of other work.

### The hosted service

- **Accounts** ([ADR 0030](0030-hosting-aws.md)): an organisation-wide **CloudTrail** trail in every account, **GuardDuty** threat detection in every account (with runtime monitoring for the production containers), and **IAM Access Analyzer**. AWS Security Hub waits until there is a reason for its cost.
- **People:** sign-in only through IAM Identity Center, with MFA, and no long-lived access keys for anyone. The management account's root user has MFA and is not used day to day.
- **Least privilege:** each service has its own IAM role with only the permissions it needs, defined in the CDK code alongside it.
- **Nobody connects to the production database by default.** In an emergency, break-glass access through IAM Identity Center is time-limited, raises an alert, and is recorded. Data fixes are scripts reviewed in a pull request and run as jobs, never typed into a live database.
- **Support** works through the API, with the operations user's audited self-add to the business's account ([ADR 0004](0004-multi-tenant.md)), never by reading the database.
- **An incident response runbook** is written before the first paying business: who decides, how to contain, how to tell affected businesses. Notification under Australia's Notifiable Data Breaches scheme is decided in the privacy ADR.

## Options considered

### Standard

1. **OWASP ASVS Level 2**: a public, testable checklist that suits a business application with sensitive data; reviewers and penetration testers know it. **Chosen.**
2. **OWASP Top 10 only**: a list of common risks, not a standard to verify against.
3. **A certification such as ISO 27001 or SOC 2**: expected by large customers; far too heavy for a one-maintainer project now. It may be needed later for larger businesses.

### Cross-site request forgery

1. **`SameSite=Lax` cookies with Fetch Metadata, `Origin` and JSON-only checks**: nothing for the client to manage, and supported by every current browser. **Chosen.**
2. **Anti-forgery tokens** (ASP.NET Core's antiforgery): the traditional defence; another token for the single-page app to fetch, store and send, for no extra protection in modern browsers.
3. **`SameSite=Strict`**: stronger in principle; adds little on top of these checks, because `GET` requests never change anything, and the sign-in flow's short-lived cookies must be `Lax` anyway to survive the redirect back from the identity service once businesses have custom domains.

### Minimum TLS version for viewers

1. **TLS 1.3 only**: modern ciphers only, quantum-safe key exchange at CloudFront, nothing to break because there are no integrators yet, and easy to loosen. **Chosen**, by Stuart.
2. **TLS 1.2 minimum**: works with older integration clients, such as .NET Framework on Windows Server 2019 and earlier; keeps older ciphers in use, and is hard to tighten once integrators depend on it.

### Business secrets

1. **Envelope encryption in the application with a KMS key**: a database dump or backup reveals no secrets, and KMS records every use of the key. **Chosen.**
2. **Store them in Secrets Manager, one secret each**: strong; about US$0.40 a secret a month, which grows with every business and webhook.
3. **Rely on database encryption at rest**: protects the disks, not a dump or a leaked backup.

## Consequences

- The design doc template's security section now asks for a threat model, and reviews check it.
- ADR 0030's costs grow by about **A$15 a month**: GuardDuty threat detection with runtime monitoring in production (about A$10), GuardDuty in the other accounts, and a KMS key per environment (US$1 a month each). CloudTrail's first copy of management events is free. Total running costs at launch become about A$340 a month, close to the first budget alert at A$350.
- The repository needs a `SECURITY.md`, and its GitHub settings need private vulnerability reporting, secret scanning with push protection, and Dependabot security updates turned on.
- The front-end design confirms the Content Security Policy against Mantine, and the deployment design writes the CloudFront Function and the TLS policies.
- The penetration test and incident runbook are tasks before the first paying business, alongside the Multi-AZ database ([ADR 0030](0030-hosting-aws.md)).
- Every new dependency is a small decision recorded in its pull request, which slows adding packages slightly, deliberately.
