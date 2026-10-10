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

- Candidate pattern: a transactional email service sends; delivery, bounce and complaint events are received by webhook or notification topic and written back to the record (e.g. Amazon SES with SNS, Postmark, Resend). Provider not yet decided.
- Domain authentication (SPF, DKIM, DMARC) on a dedicated sending subdomain is mandatory.
- Provider sits behind an interface so self-hosters can choose.

## Design

_To do._

## Failure modes

_To do._

## Alternatives considered

_To do._

## Open questions

_To do._
