# 0011. Documentation before implementation

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-10 |
| **Deciders** | Stuart Meeks |

## Context

[ADR 0002](0002-docs-as-code.md) decided where documentation lives. This record decides when it is written. Signboard is built mostly with AI assistance in short, irregular sessions. Without a written plan, each session starts from memory and guesswork, and an AI assistant will happily build the wrong thing quickly. The project also intends to publish an honest account of how it was built with AI, which depends on the planning being written down.

## Decision

Nothing is built until it is documented. Before code for a feature is written:

1. The **problem** is documented: user stories with Given/When/Then acceptance criteria in a PRD.
2. The **proposed implementation** is documented: a design doc covering the data model, the API contract ([ADR 0010](0010-api-first.md)), state changes, failure modes and alternatives.
3. Any significant decision gets an ADR.
4. All three are reviewed and set to **Approved** (ADRs to **Accepted**) before implementation starts.

Every implementation pull request links the stories and design doc it implements. If building something shows that the documentation is wrong, the documentation is corrected in the same pull request, so the docs always describe what the code does.

Bug fixes need an issue that describes the problem, its cause and the intended fix, rather than a full design doc, unless the fix changes designed behaviour.

## Options considered

1. **Documentation before implementation, always**: slower to start; far less rework, and the docs stay a reliable source of context for people and AI alike. **Chosen.**
2. **Documentation alongside implementation**: faster; in practice docs trail behind the code and decisions go unrecorded.
3. **Documentation after implementation**: describes what was built rather than what was intended; rarely happens at all.

## Consequences

- Early progress is measured in approved documents, not working software.
- Small changes still need a short written problem and plan, which adds friction that must be kept proportionate.
- The docs and their Git history become the raw material for a separate "how we built this with AI" site.
