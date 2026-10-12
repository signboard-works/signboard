# 0023. Email sending

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

Unreliable customer email is one of the main reasons the design partner shop wants to leave its current system: quotes and invoices land in junk folders, and nothing the shop or the vendor's support tried has fixed it. The cause of that kind of problem is almost always authentication: email that claims to come from a business's domain without being signed and aligned for that domain.

Email now matters for more than documents. Sign-in links and codes are sent by email ([ADR 0021](0021-authentication.md)), so if email stops arriving, people cannot sign in. Signboard is multi-tenant ([ADR 0004](0004-multi-tenant.md)), so one business's bad sending must not harm every other business's email, or anyone's sign-in.

## Decision

### Two streams

- **Business email:** quotes, proofs, invoices and notifications sent on behalf of a business.
- **Platform email:** sign-in links and codes, invitations and security alerts, sent by Signboard itself from `signboard.works`.

The two streams use separate sending configurations, so a business's sending reputation can never affect platform email.

### Sending domains

- **A business sends from its own domain once it is verified.** The business adds DNS records for a sending subdomain (for example `mail.example.com`): DKIM keys and a custom return-path domain, so that SPF and DKIM both align with its domain for DMARC. Email is then from the business's own address, such as `quotes@example.com`.
- **Until then, email falls back to a Signboard sending domain** with the business's name: `"Example Signs via Signboard" <example-signs@send.signboard.works>`, with Reply-To set to the business's real address. A new business can send on its first day; verifying its domain improves deliverability and is encouraged in the app.
- **Replies go to the business's own mailbox.** Signboard never receives email.

### Reliable, recorded sending

- An email is stored in the **same transaction** as the change that caused it (an outbox), then sent by a background worker with retries. "Quote sent" and "email queued" can never disagree.
- Every email is an object in the `notifications` module ([ADR 0020](0020-url-structure.md)), with its own ID ([ADR 0017](0017-object-ids.md)).
- Delivery events (accepted, delivered, delayed, bounced, complained) are received from the provider and written onto the email's record. The quote, proof or invoice shows whether its email was delivered.
- **Suppression lists:** a hard bounce or a complaint stops further email to that address, and staff can see why. There is a list per business and a global list for Signboard.
- **No open or click tracking.** Whether a customer looked at a quote comes from the customer portal, which is a fact rather than a guess ([ADR 0020](0020-url-structure.md)).

### Content

- Quotes and invoices are sent with **both** a PDF attachment and a link to view (and, for quotes, approve) them online. The link carries no credentials ([ADR 0021](0021-authentication.md)).
- Templates are per-business configuration ([ADR 0006](0006-configuration-over-customisation.md)): branded, plain and mostly text. Every email has a plain-text part. How templates are written and rendered is decided in [ADR 0048](0048-documents-and-templates.md).

### Providers

- The provider sits behind an interface.
- **Hosted service:** Amazon SES in the Sydney region (`ap-southeast-2`), with events delivered through Amazon SNS to an SQS queue that the worker reads ([ADR 0034](0034-background-work.md)).
- **Self-hosted:** Amazon SES or plain SMTP. SMTP sends but cannot report delivery events, and the app says so plainly wherever delivery status would appear.

## Options considered

1. **Own domain when verified, Signboard fallback until then; SES with events; separate platform stream**: fixes the authentication problem at its root without blocking new businesses; delivery is visible on every document. **Chosen.**
2. **Require a verified domain before sending**: guarantees alignment; blocks a new business until its DNS is done, which may take days for a small shop.
3. **Always send from a Signboard domain**: no DNS work for businesses; email never comes from the business itself, which customers trust less and filters score lower.
4. **Postmark or Resend**: excellent deliverability and simpler event webhooks; another vendor relationship, and less control over Australian data residency.
5. **Open and click tracking**: familiar "opened" indicators; unreliable since mail apps preload images, a privacy cost, and rewritten links hurt deliverability.

## Consequences

- The app needs a domain verification flow: show the DNS records, check them, and report what is missing in plain language.
- Platform email is part of signing in, so its delivery is monitored like any other critical dependency.
- `send` is reserved as a subdomain name ([ADR 0020](0020-url-structure.md)).
- Moving to SES production sending needs AWS approval and attention to bounce and complaint rates, which suppression lists keep low.
- Self-hosters on SMTP get sending without delivery status, and must manage their own domain authentication.
