# PRD: <Module name>

| | |
|---|---|
| **Status** | Draft |
| **Owner** | <name> |
| **Last updated** | YYYY-MM-DD |
| **Related** | <links to design docs, ADRs, other PRDs> |

## Summary

One paragraph: what this module is and why it matters to a made-to-order business, starting with the design partner shop.

## Problem

What goes wrong today, for whom, and how often. Use real examples from the shop floor where possible.

## Goals

- What this module must achieve.

## Non-goals

- What this module deliberately does not do (and where that need is handled instead, if anywhere).

## Users

Which personas from the [vision](../../vision.md#personas) use this, and in what situations.

## User stories

Each story has a stable ID so it can be referenced from design docs, issues and tests.

### <PREFIX>-001: <Short title>

**As a** <persona>, **I want** <capability>, **so that** <outcome>.

**Acceptance criteria**

```gherkin
Given <starting state>
When <action>
Then <observable result>
```

**Notes:** edge cases, examples, open points.

## Business rules

Numbered rules that apply across stories (e.g. "BR-1: A sent quote cannot be edited; it must be revised as a new version").

## States and lifecycle

Statuses this module's main record moves through, and what triggers each transition. A Mermaid state diagram is encouraged.

## Dependencies

Other modules this one needs, and which delivery phase it belongs to.

## Open questions

- [ ] Questions still to be answered, with who can answer them.
