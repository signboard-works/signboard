# 0024. Product analytics

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

To improve Signboard we need to know two things: how it is used (which features, where people get stuck, where customers drop off between opening and approving a quote), and how people feel about it (satisfaction surveys such as NPS). Cost matters: free is best.

Signboard's screens are full of business data: customer names, prices, margins. None of that may leave Signboard for analytics. Self-hosted installations belong to their operators ([ADR 0004](0004-multi-tenant.md)) and must not send data anywhere they have not chosen. Everything about the project is open ([ADR 0013](0013-work-in-the-open.md)), including what is measured.

## Decision

We will use **PostHog Cloud** (open source, MIT) on its free tier, for both product analytics and in-app surveys.

### Who is tracked

- **Everyone who uses the app:** business staff, customer users in the customer portal, and operations users.
- Users are identified by their object ID (`USR-…`, [ADR 0017](0017-object-ids.md)), never by name or email address. Each account (`ACC-…`) is a PostHog group, so usage can be compared by business and by account kind.
- A user can opt out of analytics in their own settings.

### What is tracked

- **Page views and named events only.** Events are named for what happened, such as `quote_sent`, `job_moved` or `grid_filtered`, and are defined in each feature's design. Automatic capture of every click is off.
- **No business data in events.** Event properties are limited to object IDs, kinds and states, counts and durations. Never free text, names, contact details, prices or any field outside the actor's general field groups ([ADR 0022](0022-authorisation-and-roles.md)).
- **Session replay is off at launch.** If it is turned on later, all on-screen text and every input are masked.
- **Surveys** (for example NPS) run in the app through PostHog. What is asked, of whom and how often is decided in the relevant design doc.

### How it is sent

- Events go through a path on Signboard's own hosts (`/ingest`), which the edge router forwards to PostHog ([ADR 0015](0015-separate-deployments-one-origin.md)), so analytics stays within Signboard's domains.
- Data is held in PostHog's EU region. It is anonymous usage data with no business data, so it does not need to stay in Australia, unlike email ([ADR 0023](0023-email-sending.md)).
- **Self-hosted installations send nothing by default.** An operator can point their installation at their own PostHog project in configuration.
- What is collected and why is disclosed in the privacy policy and in these docs.

## Options considered

1. **PostHog Cloud, named events, no business data, replay off**: free at Signboard's scale, open source, analytics and surveys in one tool. **Chosen.**
2. **Mixpanel or Amplitude**: strong analytics with free tiers; closed source, and surveys need another tool.
3. **Heap**: automatic capture of everything; limited free tier, and capturing everything is exactly what Signboard must avoid.
4. **Plausible or Umami**: open source and simple; website traffic only, nothing about how the app is used.
5. **Self-hosted PostHog**: data under our control; heavy to run and discouraged by PostHog at small scale.
6. **No analytics**: nothing to manage; product decisions made blind.

## Consequences

- Each feature's design lists its analytics events, and reviews check that no event carries business data.
- Free-tier limits must be watched. Exceeding them means choosing between paying, sampling or measuring less.
- Ad blockers may still block some events. The numbers are a guide, not an exact count.
- Turning session replay on later is its own decision, with masking verified before release.
