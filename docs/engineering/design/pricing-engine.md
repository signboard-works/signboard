# Design: Pricing engine

| | |
|---|---|
| **Status** | Draft |
| **Author** | Stuart Meeks |
| **Last updated** | 2026-10-10 |
| **Phase** | 1 |

## Context

How a quote line's price is calculated. The most important design in the system.

## Known constraints (to be expanded)

- Pricing methods: area, length, machine or cutting time, per unit with quantity breaks, labour or design hours, fixed.
- A single line can combine several components, e.g. an illuminated acrylic sign is material plus CNC time plus fabrication plus installation.
- Rates, markups, minimums and rounding are configuration. Arithmetic, precision, units and tax follow [ADR 0036](../decisions/0036-money-tax-and-time.md); commercial rounding (such as to the nearest $5) is decided here.
- Every calculated price must be explainable: staff can see how it was built up.

## Design

_To do._

## Failure modes

_To do._

## Alternatives considered

_To do._

## Open questions

_To do._
