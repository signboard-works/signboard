# 0021. Authentication

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Users are global and identified by email address, and every request acts as one of a user's memberships ([ADR 0010](0010-api-first.md)). Each business has its own host, which serves the app and the API from one origin; the browser signs in with an HttpOnly session cookie on that origin, and integrations use bearer tokens ([ADR 0010](0010-api-first.md), [ADR 0015](0015-separate-deployments-one-origin.md), [ADR 0020](0020-url-structure.md)).

So one person may need to use several hosts: a designer working for two businesses, or a club's office manager approving quotes from two shops. They should sign in once. The same code must also run self-hosted, without depending on an outside identity service ([ADR 0004](0004-multi-tenant.md)).

Customer users rarely sign in, and usually only because an email has just arrived. Staff on the shop floor often share a tablet.

## Decision

### One identity service, single sign-on across businesses

- A central identity service at `id.signboard.works` handles sign-in for every user and every host, using **OpenID Connect** (authorisation code flow with PKCE).
- Each business's host is a client of the identity service. The API on that host completes the sign-in and sets that origin's own secure, HttpOnly session cookie. Browser code never handles tokens.
- Signing in once signs the user in everywhere: moving to another business's host redirects through the identity service without asking again.
- After sign-in, the user chooses the account to act in, or goes straight to it if they have only one. Switching accounts is a session action ([ADR 0020](0020-url-structure.md)).
- The identity service is built in-house with **ASP.NET Core Identity** (MIT) for users and credentials, and **OpenIddict** (Apache 2.0) as the OpenID Connect server. A self-hosted installation runs the same service on its own host.

### How people sign in

- **Passkeys** are the preferred method.
- **An emailed one-time link or code** is always available.
- **Passwords** are allowed as an option, but a password alone is never enough: signing in with a password always needs a second factor, from an authenticator app or a passkey. Passwords follow NIST SP 800-63B: a minimum length, a check against known-breached passwords, no composition rules and no forced expiry.
- **Operations users and business administrators** must use a passkey or a second factor whatever their method. An emailed link alone is not enough for them.
- An email address must be verified before any membership takes effect.
- Sign-in with Google or Microsoft accounts is a later decision.

### Links in emails

A link in an email to a customer opens the object it is about ([ADR 0020](0020-url-structure.md)) but carries no credentials. If the recipient is not signed in, they are asked to sign in, normally with an emailed code. A forwarded email therefore gives nobody access.

### Sessions

- One sign-in per device is acceptable for now, including shared tablets on the shop floor. Quick switching between users on a shared device is a later decision.
- Operations users' sessions are short-lived. Session lifetimes are set in the authentication design.
- Users can see and end their own sessions; business administrators can end their members' sessions.

### Integrations

- An **API client** belongs to one business or customer account ([ADR 0010](0010-api-first.md)) and holds one or more **API tokens**.
- A token has a fixed set of permissions and an expiry date, is shown once when created, is stored only as a hash, and can be revoked.
- Creating, revoking and using tokens is audited ([ADR 0019](0019-audit-trail.md)).
- OAuth client credentials can be added later if integrators ask for them.

## Options considered

1. **In-house identity service on ASP.NET Core Identity and OpenIddict**: no per-user cost, no outside dependency for self-hosters, standard OpenID Connect at the boundary. Signboard owns security-critical code. **Chosen.**
2. **Keycloak** (Apache 2.0): mature and full-featured; a separate Java service to run, upgrade and back up beside Signboard.
3. **Hosted identity service** (Auth0, Microsoft Entra External ID): little to build; per-user cost, and self-hosters would need a different answer.
4. **Duende IdentityServer**: the best-known .NET option; needs a commercial licence above a revenue threshold.
5. **Separate sign-in on each business's host**: no central service; users in several businesses sign in repeatedly, and credentials cannot be shared.

## Consequences

- Business hosts talk to the identity service only through standard OpenID Connect, so the service can be replaced by Keycloak or a hosted provider later without changing the rest of Signboard.
- Owning the identity service means owning its security: it gets its own design doc, the strictest tests, and an independent security review before the first paying business.
- Email delivery becomes part of signing in, not just of sending quotes. It must be fast and reliable ([email delivery design](../design/email-delivery.md)).
- Passkeys are tied to a domain. The authentication design must choose that domain so passkeys work on every business's host, and for self-hosted installations.
- Sign-ins, failed sign-ins, MFA changes and session ends are security events in the audit trail ([ADR 0019](0019-audit-trail.md)).
