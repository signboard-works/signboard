# Design: <Component name>

| | |
|---|---|
| **Status** | Draft |
| **Author** | <name> |
| **Last updated** | YYYY-MM-DD |
| **Implements** | <PRD links and story IDs> |
| **Related ADRs** | <links> |

## Context

What this component is responsible for, and which requirements drive it.

## Goals and non-goals

- **Goals:** what the design must achieve technically.
- **Non-goals:** what it deliberately leaves out.

## Design

The proposal itself. Diagrams (Mermaid) are encouraged: components, sequences, entity relationships, state machines.

## Data

Entities, fields and relationships this component owns or touches. Link to the [data model](../design/data-model.md) rather than duplicating it.

## Interfaces

APIs, events, and integration points, with example payloads.

## Failure modes

What can go wrong (network, third-party outages, bad data) and how the design responds.

## Security and privacy

Authentication, authorisation, secrets, and personal data handled. Where the design crosses a trust boundary (customer access, files, webhooks, integrations, anything a business can configure), a short threat model: what could go wrong, and what prevents it ([ADR 0039](../decisions/0039-security-baseline.md)).

## Alternatives considered

Other approaches and why they were not chosen. Significant choices should also get an ADR.

## Open questions

- [ ] Unresolved technical questions.
