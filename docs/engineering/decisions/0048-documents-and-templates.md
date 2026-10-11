# 0048. Documents and templates

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

> **Draft for review.** Written from Claude's recommendations. The choices still open are listed under [Questions for review](#questions-for-review).

## Context

Signboard produces documents for each business: quotes, sales orders, invoices and job sheets as PDFs, and the emails that carry them ([ADR 0023](0023-email-sending.md)). Every business wants them to look like its own, with its logo, colours, wording and terms, and how they look is configuration, never code ([ADR 0006](0006-configuration-over-customisation.md)).

Templates written by businesses are code written by untrusted people that runs on our servers. A careless template engine lets one business read another's data, read server files, or tie up the worker with an endless loop. The documents must also meet decisions already made:

- the recipient sees only what they may see ([ADR 0010](0010-api-first.md));
- text in the recipient's language, formats from the business's locale, and dates with the month as a word or in ISO 8601 ([ADR 0037](0037-localisation.md));
- amounts, tax and rounding exactly as calculated by the API ([ADR 0036](0036-money-tax-and-time.md));
- every PDF tagged for screen readers to PDF/UA, and brand colours contrast-checked ([ADR 0042](0042-accessibility.md));
- a sent document kept exactly as sent, as an immutable file ([ADR 0027](0027-file-storage.md));
- heavy work in background jobs, never in an API request ([ADR 0034](0034-background-work.md), [ADR 0035](0035-retries-and-resilience.md));
- libraries under allowed licences ([ADR 0039](0039-security-baseline.md#supply-chain)), and initial development kept as cheap as possible.

## Decision

### Templates: Liquid, rendered by Fluid

- Document and email templates are written in **Liquid**, the template language used by Shopify and many other products, so it is documented widely and familiar to many people who design for the web.
- They are rendered by **Fluid** (MIT, [sebastienros/fluid](https://github.com/sebastienros/fluid), active), which renders Liquid in .NET.
- Fluid's own guidance says it "is a template engine, not a security sandbox", and lists the controls an application must apply to render user-provided templates safely. We apply all of them:
    - **a fixed view model per document type** (see below), never domain entities, services or configuration;
    - **limits on everything**: template length before parsing, recursion, steps (`MaxSteps`), output size (`MaxOutputSize`), collection size, and a per-render deadline;
    - **no file access**: the default null file provider, or a provider scoped to the business's own templates if partials are needed;
    - **only our own filters**, such as `money`, `date` and `quantity`, which follow ADRs 0036 and 0037. Templates never format numbers or dates themselves;
    - **HTML encoding on by default**. Any HTML a business writes, such as terms and conditions, is cleaned with HtmlSanitizer (MIT, [ADR 0039](0039-security-baseline.md#safe-code)) when saved.

### What a template can see

- Each document type has one **documented view model**: the fields a template may use, such as the quote's lines, totals, customer and the business's details. It is built for the recipient, so a customer's quote never contains fields hidden from that customer, such as cost prices ([ADR 0010](0010-api-first.md)). It is the same data the recipient could see through the API.
- The view model is versioned with the API. Adding fields is safe; removing one follows the API's rules for breaking changes ([ADR 0029](0029-api-errors-idempotency-versioning.md#versioning)).
- Text produced by Signboard, such as labels like "Quantity" and "Total", comes from translation keys in the recipient's language ([ADR 0037](0037-localisation.md)). Text the business writes is shown as written.

### Customising, in two levels

1. **Settings, for everyone.** Logo, brand colour, font from a short list, terms and conditions, footer text, and which optional sections to show. Most businesses need nothing more, and these cannot break a document.
2. **Template source, for those who want it.** A business administrator can edit the Liquid template itself, with a permission of its own ([ADR 0022](0022-authorisation-and-roles.md)).

Signboard ships a **default template for each document type and country**. A business starts from the default and can always reset to it.

### Required content

Some content is required by law and cannot be removed. The Australian default tax invoice meets the ATO's requirements ([Tax invoices](https://www.ato.gov.au/businesses-and-organisations/gst-excise-and-indirect-taxes/gst/tax-invoices)): the words "Tax invoice", the seller's identity and ABN, the date of issue, a description of what was sold, the GST-inclusive price, and the GST amount or a statement that the total includes GST; and, for sales of A$1,000 or more including GST, the buyer's identity or ABN. These come from a locked block that a template can place but not remove, and a template that leaves it out is refused when saved. Each country's defaults list their own required content.

### Saving, previewing and versions

- A template is checked when it is saved: it must parse, and render within the limits against sample data. Errors are shown with the line they occur on.
- **Preview** renders the template against sample data, or against a real draft document, before it is used.
- Templates are **versioned**: saving creates a new version, and the old versions are kept.
- **A failed render fails the send.** If a template fails when a document is sent, the job fails and the business is told. Signboard never silently sends the default template instead, because the customer would receive something the business did not choose.

### PDFs: HTML and CSS, printed by Chromium

- A PDF is made by rendering the template to HTML and CSS, then printing it with **headless Chromium** through **Playwright for .NET** (MIT, [microsoft/playwright-dotnet](https://github.com/microsoft/playwright-dotnet)). Emails and PDFs share one way of writing templates, and print CSS handles page size, margins, headers and page numbers. Paper size comes from the country ([ADR 0037](0037-localisation.md)).
- **Tagged for screen readers.** Playwright's PDF options include `Tagged` ("whether or not to generate tagged (accessible) PDF") and `Outline`, both added in version 1.42 ([Page.PdfAsync](https://playwright.dev/dotnet/docs/api/class-page#page-pdf)). Both are always on. Chromium builds the tags from the HTML, so the default templates use real headings, lists and table markup, and the document's title and language are set.
- **Checked against PDF/UA.** Chromium does not claim PDF/UA conformance for its tagged output, so CI validates the PDFs produced by every default template with **veraPDF**, the PDF Association's open-source validator (dual-licensed GPL 3 or MPL 2.0, used as a test tool and never shipped). If Chromium's output cannot pass, the fallback is to build PDFs with **PDFsharp** (MIT), which can produce PDF/UA documents through its `UAManager` ([PDFsharp accessibility](https://docs.pdfsharp.net/PDFsharp/Topics/PDF-Features/Accessibility.html)), at the cost of a second way of laying out documents.
- **Locked down.** Chromium renders only our own HTML: JavaScript is off, and every network request is refused, so a template cannot fetch anything. Images, such as the business's logo, are passed in as embedded data rather than URLs.
- **Fonts** are embedded: Inter (SIL Open Font Licence) by default, with Noto Sans (SIL Open Font Licence) as a fallback that covers almost every script. A business chooses from a short list. Uploaded fonts are not supported at launch, because their licences are hard to check.

### Where rendering runs

- Rendering is a **background job** ([ADR 0034](0034-background-work.md)), with a per-type concurrency limit, so a burst of invoices cannot swamp the worker.
- At first it runs in the **existing worker service**, given about 1 GB more memory for Chromium: about US$3 a month on Fargate ARM in Sydney (US$0.00426 per GB-hour), or about A$5 including GST. If PDF rendering slows other jobs, it moves to a worker pool of its own, which ADR 0034 already allows. A separate always-on task of 0.5 vCPU and 1 GB would cost about US$17 (A$27) a month, so it waits until it is needed.
- The worker image needs Chromium and its libraries, so it cannot use the chiseled base image that [ADR 0039](0039-security-baseline.md#supply-chain) sets for the other services. It still runs as a non-root user, with only a temporary directory writable.
- Self-hosted installations run the same worker image. There is no outside rendering service.

### Kept exactly as sent

When a document is sent, its PDF is stored as an immutable file ([ADR 0027](0027-file-storage.md)), together with the **template version** and a **snapshot of the view model** it was rendered from. A sent document is never regenerated. The snapshot shows exactly what data produced it, even after the quote, the customer or the template changes.

### Emails

Emails use the same engine, view models and filters. Each template produces an HTML part and a plain-text part ([ADR 0023](0023-email-sending.md)). CSS is inlined for email clients with PreMailer.Net (MIT). Emails are text and markup, never images of text ([ADR 0042](0042-accessibility.md#documents-and-emails)).

## Options considered

### Template engine

1. **Fluid (Liquid)**: MIT and active; a well-known language; documented controls for user-provided templates, including step, output and collection limits and allow-listed members. **Chosen.**
2. **Scriban** (BSD-2-Clause, active): fast and capable, with its own limits, and it can also read Liquid; its own language is less familiar, and supporting two languages helps nobody.
3. **Handlebars.Net** (MIT): logic-less and simple; too limited for documents with conditional sections and grouped lines, and has fewer controls for untrusted templates.
4. **Razor, through RazorLight** (Apache 2.0, little activity since 2024): full C# in templates; runs arbitrary code, so it is never safe for templates written by businesses.

### PDF rendering

1. **HTML and CSS printed by headless Chromium through Playwright**: one way of writing templates for emails and PDFs; full CSS layout; tagged output and outlines built in; free. A heavy dependency, and its PDF/UA conformance must be checked by us. **Chosen.**
2. **PDFsharp and MigraDoc** (MIT): light, pure .NET, and can produce PDF/UA; documents laid out in code rather than HTML, so businesses could not edit them as templates. Kept as the fallback.
3. **QuestPDF**: an excellent code-first library; its Community Licence is, in its own words, "a source-available commercial license", free only below US$1 million revenue and not for public-sector or listed organisations. [ADR 0039](0039-security-baseline.md#supply-chain) rules out source-available licences, and self-hosters could be caught by it.
4. **iText**: AGPL or commercial. Never.
5. **A hosted PDF service**: no heavy dependency; a cost per document, business data sent to another company, and nothing for self-hosters.

## Consequences

- Each document type gets a view model, default templates for each supported country, and sample data, specified in the design for its module. Templates become objects with an ID prefix, registered in the [data model](../design/data-model.md).
- CI renders every default template, checks it with veraPDF, and compares it with a reference image to catch layout changes.
- The worker image includes Chromium, and adds about A$5 a month in memory. Chromium updates arrive through Dependabot with Playwright.
- The template editor needs syntax errors shown on the line, a preview, version history and reset to default. Its design belongs to the settings design.
- A business that edits template source can produce an ugly document, but not an unsafe one, and never one missing required content.

## Questions for review

1. **Liquid, rendered by Fluid**, for its familiarity and documented controls for untrusted templates. Or Scriban?
2. **Two levels of customisation**, settings for everyone and template source for administrators. Or settings only at launch, with source editing later?
3. **Chromium through Playwright for PDFs**, with veraPDF checking PDF/UA in CI and PDFsharp as the fallback. Or start with PDFsharp, giving up editable templates for PDFs?
4. **Rendering inside the existing worker** for about A$5 a month in memory, moving to its own pool only when needed. Agreed?
5. **A failed render fails the send**, rather than falling back to the default template. Agreed?
6. **Required legal content in locked blocks** that a template cannot remove. Agreed?
7. **No uploaded fonts at launch**, only a short list of openly licensed fonts. Agreed?
