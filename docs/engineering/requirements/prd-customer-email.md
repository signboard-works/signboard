# PRD: Customer email

| | |
|---|---|
| **Status** | Draft |
| **Owner** | Stuart Meeks |
| **Last updated** | 2026-10-10 |
| **Phase** | 2 |

## Summary

Send quotes, proofs and invoices to customers reliably and professionally.

## Known requirements (to be expanded)

- Emails send from the business's own authenticated domain (SPF, DKIM, DMARC aligned), via a sending subdomain, once verified. Until then they send "via Signboard" with Reply-To set to the business ([ADR 0023](../decisions/0023-email-sending.md)).
- Quotes and invoices go out with a PDF attachment and a link to view them online.
- Bounces, complaints and deliveries are recorded against the customer and document, so staff can see when an email did not land.
- Templates are plain, branded and configurable. See [email delivery](../design/email-delivery.md).

## Problem

_To do._

## Goals

_To do._

## Non-goals

_To do._

## User stories

Story IDs use the prefix `EML-`.

_To do._

## Business rules

_To do._

## States and lifecycle

_To do._

## Open questions

See [open questions](open-questions.md).
