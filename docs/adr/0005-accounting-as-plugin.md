# 0005. Accounting systems behind a plugin interface

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

The design partner shop is believed to use Xero. Other shops use MYOB, QuickBooks or others. Coupling invoicing directly to one provider would make Signboard unusable for most shops.

## Decision

We will define an accounting interface covering contacts, invoices and payments, and implement Xero as the first provider. Signboard's invoicing works without any provider configured.

## Consequences

- Other providers can be contributed without touching core invoicing.
- The interface must be designed around the common denominator of providers, not Xero's specifics.
