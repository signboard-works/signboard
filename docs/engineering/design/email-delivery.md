# Design: Email delivery

| | |
|---|---|
| **Status** | Draft |
| **Author** | Stuart Meeks |
| **Last updated** | 2026-10-10 |
| **Phase** | 2 |

## Context

Sending customer email reliably and tracking what happened to it.

## Known constraints (to be expanded)

- Decided in [ADR 0023](../decisions/0023-email-sending.md): business and platform streams kept separate; businesses send from their own verified domain (DKIM and custom return path on a sending subdomain, aligned for DMARC), falling back to `send.signboard.works` with Reply-To until verified.
- Outbox in the same transaction, background sending with retries; every email an object with delivery events written back; suppression lists per business and global; no open or click tracking.
- Amazon SES in `ap-southeast-2` with events via SNS and a queue; plain SMTP supported for self-hosters, without delivery events.
- Quotes and invoices sent with a PDF attachment and a link that carries no credentials.

## Design

_To do._

## Failure modes

_To do._

## Alternatives considered

_To do._

## Open questions

_To do._
