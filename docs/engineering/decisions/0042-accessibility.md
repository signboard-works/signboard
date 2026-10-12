# 0042. Accessibility

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

## Context

Signboard is used in an office, on a tablet on the shop floor, on a screen on the workshop wall, and by customers on their phones approving a quote or a proof. Its users include people who use a keyboard rather than a mouse, a screen reader, magnification, or who simply have gloves on or are reading from across a room.

[ADR 0009](0009-frontend-react.md) already sets the target, WCAG 2.2 AA, and requires a keyboard alternative to every drag-and-drop action and touch support for grids and the kanban board. What is missing is how that target is met and kept: which devices and browsers, what documents and emails must do, and how it is tested. Like localisation ([ADR 0037](0037-localisation.md)), accessibility is cheap from the first screen and expensive to retrofit.

In Australia, the Disability Discrimination Act 1992 applies to services like Signboard. The Australian standard for accessible technology, AS EN 301 549:2024, still incorporates WCAG 2.1, and the Australian Human Rights Commission's guidelines advise using WCAG 2.2 where appropriate ([standards and guidelines on digital accessibility](https://humanrights.gov.au/our-work/disability-rights/chapter-3-standards-and-guidelines-digital-accessibility)). WCAG 2.2 AA meets both.

## Decision

### The standard

- **WCAG 2.2 Level AA** for everything people use: the staff app, the customer portal, the identity service's sign-in pages ([ADR 0021](0021-authentication.md)), emails and the documentation site.
- **Two AAA criteria as well**, because they matter on the shop floor:
    - **touch targets of at least 44 by 44 pixels** (AA asks for 24);
    - **no animation when the device asks for reduced motion**, including kanban cards moving with live updates ([ADR 0032](0032-events-webhooks-live-updates.md)).
- Other AAA criteria are not targets.

### Documents and emails

- **Every PDF Signboard generates is tagged for screen readers**, whether it is sent to a customer, such as a quote or invoice, or used internally, such as a job sheet. Tagging meets PDF/UA (ISO 14289): reading order, headings, table structure and alternative text for images. How PDFs are produced, and how they are checked against PDF/UA, is decided in [ADR 0048](0048-documents-and-templates.md#pdfs-html-and-css-printed-by-chromium).
- **Emails** use semantic HTML, with a plain-text part ([ADR 0023](0023-email-sending.md)), real text rather than text in images, and alternative text for logos.

### Devices

| Device | Who | What it must do |
|---|---|---|
| **Desktop** | Office staff | Everything |
| **Tablet** | The shop floor | Everything a production user does, by touch, in landscape and portrait |
| **Wall screen** | The workshop | A read-only kanban view: large type, readable from across a room, kept current by live updates, with no interaction needed |
| **Phone** | Customers; staff looking something up | The customer portal works fully, because customers approve quotes and proofs on their phones. Staff screens reflow to 320 pixels wide, as WCAG requires, but are not designed for phone use |

These are confirmed with the design partner shop through [open question 13](../requirements/open-questions.md).

### Browsers

- The **last two major versions** of Chrome, Edge, Firefox and Safari, including Safari on iPad and iPhone.
- No Internet Explorer and no legacy builds. One `browserslist` setting drives the build and the polyfills.

### Built into the components

Accessibility is the default in Signboard's wrapper components ([ADR 0009](0009-frontend-react.md)), so screens get it without effort and cannot easily lose it.

- **Names and labels are required.** Inputs need a label, and icon-only buttons need an accessible name, enforced by their TypeScript types.
- **Errors.** Field errors from the API ([ADR 0029](0029-api-errors-idempotency-versioning.md)) are shown against their fields and linked to them for screen readers, with a summary at the top of the form that moves focus to the first error.
- **Focus** is always visible, moves sensibly when dialogs open and close, and is never trapped.
- **Colour is never the only signal.** A kanban card's status, an overdue invoice or a failed sync also has text or an icon.
- **Live updates are announced politely.** Changes from other people ([ADR 0032](0032-events-webhooks-live-updates.md)) are announced to screen readers without interrupting, and are grouped rather than read out one by one.
- **Grids** follow the WAI-ARIA grid pattern for keyboard navigation and announce sorting, filtering and paging.
- **Text and zoom.** Pages work at 200% zoom and with larger text settings, using relative units.
- **Time limits** warn before a session expires and allow it to be extended.

### Drag and drop has an alternative

- Every move on the kanban board, and anywhere else something is dragged, can also be made **without dragging**: by a "Move to…" menu, and by the keyboard. This meets WCAG 2.2's criterion on dragging movements, and is the same action through the same API ([ADR 0031](0031-concurrency.md)).
- The drag-and-drop library must support keyboard and screen-reader dragging. dnd-kit, a candidate in [ADR 0009](0009-frontend-react.md), has both built in.

### Sign-in

- Sign-in meets WCAG 2.2's criterion on accessible authentication: passkeys and emailed codes need no memory test or puzzle, and code fields accept pasting and password managers ([ADR 0021](0021-authentication.md)).
- **No CAPTCHA on sign-in.** If sign-ups are abused, the WAF CAPTCHA allowed by [ADR 0038](0038-rate-limiting.md) may be added to sign-up only. AWS documents its puzzles as offering audio and visual versions, screen-reader and keyboard support, and meeting WCAG ([AWS WAF CAPTCHA puzzles](https://docs.aws.amazon.com/waf/latest/developerguide/waf-captcha-puzzle.html)). That is AWS's own claim, without a stated WCAG version or an independent audit, so the sign-up page is checked with a screen reader before CAPTCHA is turned on.

### Business branding

A business's brand colour ([ADR 0006](0006-configuration-over-customisation.md)) is checked for contrast wherever it is used, in the customer portal, emails and documents. If it fails, Signboard uses a darker or lighter shade of it for text and controls, and tells the business administrator when they set it.

### Testing

- **Automated, on every pull request** ([ADR 0012](0012-testing.md), [ADR 0028](0028-branching-ci-and-releases.md)):
    - **axe-core**, through `@axe-core/playwright`, runs on every page the Playwright smoke tests visit. Any serious or critical violation fails the build. axe-core is MPL 2.0, which [ADR 0039](0039-security-baseline.md#supply-chain) allows case by case; it is a test tool, never shipped to users.
    - **eslint-plugin-jsx-a11y** (MIT) checks components as they are written.
    - Component tests find elements by role and label, as Testing Library encourages, so an element without an accessible name cannot be tested.
- **Manual**, because automated checks find only part of the problems:
    - a keyboard-only pass and a screen-reader pass, with NVDA on Windows and VoiceOver on Mac and iPad (both free), for each new screen before its pull request merges, and across the main journeys before each release;
    - a check on a real tablet for the shop-floor screens.
- **Each feature's design doc has an accessibility section**, saying how its screens meet this ADR. Acceptance criteria cover accessibility wherever a story needs more than the baseline, such as the kanban board's keyboard moves.

### Statement and reporting

- A public **accessibility statement** gives the target, the known gaps and their planned fixes, and how to report a barrier.
- Barriers are reported through the issue forms, labelled `accessibility` ([ADR 0040](0040-backlog-and-issues.md)).
- **An accessibility failure is a bug**, handled like any other, never a feature request.

## Options considered

1. **WCAG 2.2 AA, built into the components, tested automatically on every pull request and by hand for each screen**: meets the recognised benchmark, costs little when done from the start, and every tool is free. **Chosen.**
2. **WCAG 2.2 AAA throughout**: the strictest level; some criteria, such as sign language for media and the strictest contrast for every colour, are impractical for a business application. The two AAA criteria that matter here are adopted.
3. **Automated checks only**: cheaper; automated tools find only part of the problems, and miss most keyboard and screen-reader issues.
4. **Fix accessibility later**: no effort now; retrofitting every screen, document and email costs far more, and excludes people in the meantime.

## Consequences

- The front-end design doc specifies the wrapper components' accessibility rules, the grid's keyboard model and the wall-screen view. The design doc template gains an accessibility section.
- The drag-and-drop spike in [ADR 0009](0009-frontend-react.md) includes keyboard and screen-reader dragging in its pass criteria.
- [ADR 0048](0048-documents-and-templates.md) produces tagged PDFs and checks them with veraPDF in CI.
- Each screen takes a little longer to finish, for the manual checks. It is part of done, not extra.
- The accessibility statement is written before the first paying business, and kept current.
