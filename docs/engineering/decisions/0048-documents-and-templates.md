# 0048. Documents and templates

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-12 |
| **Deciders** | Stuart Meeks |

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

1. **Settings, for everyone, at launch.** Logo, brand colour, font from a short list, terms and conditions, footer text, and which optional sections to show. Most businesses need nothing more, and these cannot break a document.
2. **Template source, later.** A business administrator will be able to edit the Liquid template itself, with a permission of its own ([ADR 0022](0022-authorisation-and-roles.md)). It is built after launch, behind a release flag ([ADR 0033](0033-feature-flags.md)), because it brings an editor, version history and the full risk of templates written by businesses. The engine and every control above are built from the start, because Signboard's own templates use them, so source editing adds to the design rather than changing it.

Signboard ships a **default template for each document type and country**. A business starts from the default and can always reset to it.

### Required content

Some content is required by law and cannot be removed. The Australian default tax invoice meets the ATO's requirements ([Tax invoices](https://www.ato.gov.au/businesses-and-organisations/gst-excise-and-indirect-taxes/gst/tax-invoices)): the words "Tax invoice", the seller's identity and ABN, the date of issue, a description of what was sold, the GST-inclusive price, and the GST amount or a statement that the total includes GST; and, for sales of A$1,000 or more including GST, the buyer's identity or ABN. These come from a locked block that a template can place but not remove, and a template that leaves it out is refused when saved. Each country's defaults list their own required content.

Every rendered document is checked before it is stored or sent: the PDF's text must contain the required content. A document that fails the check is not sent, and the send fails as below.

At launch, only Signboard's own templates exist, and settings cannot change the block's styles, so it is always visible. Template source editing would let a template's CSS hide the block while its text stays in the PDF: with `display: none`, white on white, zero opacity on an ancestor, clipping or a position off the page, which no stylesheet rule of ours can reliably override. So before source editing is built, its design must keep the block out of the template's reach. The template marks where the block goes, but Signboard renders it in a reserved area of the page, outside the template's markup, and checks the rendered page that the area shows it.

### Saving, previewing and versions

- A template is checked when it is saved: it must parse, and render within the limits against sample data. Errors are shown with the line they occur on.
- **Preview** renders the template against sample data, or against a real draft document, before it is used.
- Templates are objects with the `TPL` prefix ([ADR 0017](0017-object-ids.md)), and are **versioned**: saving creates a new version under the same ID, and the old versions are kept.
- **A failed render fails the send.** If a template fails when a document is sent, the job fails and the business is told. Signboard never silently sends the default template instead, because the customer would receive something the business did not choose.

### PDFs: HTML and CSS, printed by Chromium

- A PDF is made by rendering the template to HTML and CSS, then printing it with **headless Chromium** through **Playwright for .NET** (MIT, [microsoft/playwright-dotnet](https://github.com/microsoft/playwright-dotnet)). Emails and PDFs share one way of writing templates, and print CSS handles page size, margins, headers and page numbers. Paper size comes from the country ([ADR 0037](0037-localisation.md)).
- **Tagged for screen readers.** Playwright's PDF options include `Tagged` ("whether or not to generate tagged (accessible) PDF") and `Outline`, both added in version 1.42 ([Page.PdfAsync](https://playwright.dev/dotnet/docs/api/class-page#page-pdf)). Both are always on. Chromium builds the tags from the HTML, so the default templates use real headings, lists and table markup, and the document's title and language are set.
- **Checked against PDF/UA.** Chromium does not claim PDF/UA conformance for its tagged output, so CI validates the PDFs produced by every default template, with the range of settings a business can choose, with **veraPDF**, the PDF Association's open-source validator (dual-licensed under GPL 3 or later, or MPL 2.0 or later, [veraPDF: develop](https://docs.verapdf.org/develop/); used as a test tool and never shipped). If Chromium's output cannot pass, the fallback is to build PDFs with **PDFsharp** (MIT), which can produce PDF/UA documents through its `UAManager` ([PDFsharp accessibility](https://docs.pdfsharp.net/PDFsharp/Topics/PDF-Features/Accessibility.html)), at the cost of a second way of laying out documents. Before the default templates are built, one sample invoice is printed and checked with veraPDF, so the fallback is chosen early if it is needed.
- **Templates a business edits are checked too.** A business's own template could produce poor structure, such as a table laid out with `div` elements. So, once source editing is built, every saved template version is rendered against sample data and checked with veraPDF, and a version that fails is refused with the problems listed. Running veraPDF in production means shipping it under its MPL 2.0 (or later) licence, which [ADR 0039](0039-security-baseline.md#supply-chain) allows case by case; that case is made in the source editing design.
- **Locked down.** Chromium renders only our own HTML: JavaScript is off, and every network request is refused, so a template cannot fetch anything. Images, such as the business's logo, are passed in as embedded data rather than URLs.
- **Fonts** are embedded: Inter (SIL Open Font Licence, [rsms/inter](https://github.com/rsms/inter)) by default, with Noto Sans (SIL Open Font Licence, [Noto fonts](https://notofonts.github.io/)) as a fallback. Together they cover the Latin, Greek and Cyrillic scripts at launch. Other scripts, such as Chinese, Japanese and Korean, need their own Noto fonts, which are added with the languages that need them ([ADR 0037](0037-localisation.md)). A business chooses from a short list. Uploaded fonts are not supported at launch, because their licences are hard to check.

### Where rendering runs

- Rendering is a **background job** ([ADR 0034](0034-background-work.md)), with a per-type concurrency limit, so a burst of invoices cannot swamp the worker. Each render has a deadline of 30 seconds, and a document of more than 50 pages is refused. Both limits are configuration.
- **Liquid runs in the worker.** The worker builds the view model and renders the template to HTML, under Fluid's limits.
- **Chromium runs apart from the data, once real data arrives.** Chromium's own sandbox relies on kernel features, such as user namespaces, that Fargate's containers may not provide, and Chromium in containers is commonly run with the sandbox turned off. This has not yet been tested on Fargate, so the safe assumption is that it runs without it. A Chromium bug triggered by crafted HTML or CSS would then reach whatever the process can reach, and the worker holds the database credentials for every business. So:
    - **at launch**, before any paying business, Chromium runs inside the existing worker, given about 1 GB more memory: about US$3 a month on Fargate ARM in Sydney (US$0.00426 per GB-hour, from the AWS price list of 2026-10-11 used in [ADR 0030](0030-hosting-aws.md#expected-cost)), or about A$5 including GST;
    - **before the first paying business**, Chromium moves to a **renderer** of its own: a small internal service that receives HTML and returns a PDF, with **no database access, no secrets, no AWS permissions and no outbound network**. The worker calls it as an outside service, with a timeout and retries ([ADR 0035](0035-retries-and-resilience.md)). One task of 0.5 vCPU and 1 GB costs about US$17 a month at the same prices, or about A$25 including GST, and the worker gives back its extra memory;
    - if a spike shows that Chromium's sandbox can run on Fargate after all, Chromium can stay in the worker instead, and this ADR is updated.
- The renderer's image needs Chromium and its libraries, so it cannot use the chiseled base image that [ADR 0039](0039-security-baseline.md#supply-chain) sets for the other services. It still runs as a non-root user, with a read-only root filesystem and only a temporary directory writable. While Chromium is still in the worker at launch, the worker's image has the same exception.
- Self-hosted installations run the same renderer image, as a container of their own in the Docker Compose file. There is no outside rendering service.

### Kept exactly as sent

When a document is sent, its PDF is stored as an immutable file ([ADR 0027](0027-file-storage.md)), together with the **template version** and a **snapshot of the view model** it was rendered from. A sent document is never regenerated. The snapshot shows exactly what data produced it, even after the quote, the customer or the template changes.

De-identifying a person also clears them from these snapshots, as it does from audit snapshots ([ADR 0044](0044-privacy-and-data-residency.md#erasing-personal-information)). The sent PDF is a file, so it is listed for the business to review, like any other file that may contain the person.

### Emails

Emails use the same engine, view models and filters. Each template produces an HTML part and a plain-text part ([ADR 0023](0023-email-sending.md)). CSS is inlined for email clients with PreMailer.Net (MIT). Emails are text and markup, never images of text ([ADR 0042](0042-accessibility.md#documents-and-emails)).

## Options considered

### Template engine

1. **Fluid (Liquid)**: MIT and active; a well-known language; documented controls for user-provided templates, including step, output and collection limits and allow-listed members. **Chosen.**
2. **Scriban** (BSD-2-Clause, active): fast and capable, with its own limits, and it can also read Liquid; its own language is less familiar, and supporting two languages helps nobody.
3. **Handlebars.Net** (MIT): logic-less and simple; too limited for documents with conditional sections and grouped lines, and has fewer controls for untrusted templates.
4. **Razor, through RazorLight** (Apache 2.0, little activity since 2024): full C# in templates; runs arbitrary code, so it is never safe for templates written by businesses.

### Customising

1. **Settings at launch, template source later, behind a release flag**: covers what most businesses want, and puts off the editor and the risk of business-written templates until they are needed. **Chosen.**
2. **Both from launch**: the most flexible from the start; an editor, version history and the full untrusted-template risk before anyone has asked for them.
3. **Settings only, for good**: the simplest and safest; a business that needs its own layout can never have it.

### PDF rendering

1. **HTML and CSS printed by headless Chromium through Playwright**: one way of writing templates for emails and PDFs; full CSS layout; tagged output and outlines built in; free. A heavy dependency, and its PDF/UA conformance must be checked by us. **Chosen.**
2. **PDFsharp and MigraDoc** (MIT): light, pure .NET, and can produce PDF/UA; documents laid out in code rather than HTML, so businesses could not edit them as templates. Kept as the fallback.
3. **QuestPDF**: an excellent code-first library; its Community Licence is, in its own words, "a source-available commercial license", free only below US$1 million revenue and not for public-sector or listed organisations. [ADR 0039](0039-security-baseline.md#supply-chain) rules out source-available licences, and self-hosters could be caught by it.
4. **iText**: AGPL or commercial. Never.
5. **A hosted PDF service**: no heavy dependency; a cost per document, business data sent to another company, and nothing for self-hosters.

### Where Chromium runs

1. **In the worker at launch, in a renderer with no access to data before the first paying business**: costs almost nothing while there is no real data, and a Chromium bug can never reach the database once there is. **Chosen.**
2. **In the worker for good**: about A$20 a month cheaper; Chromium without its sandbox, in the process that holds every business's data.
3. **A renderer from launch**: the same safety earlier; about A$25 a month to protect test data.

## Consequences

- Each document type gets a view model, default templates for each supported country, and sample data, specified in the design for its module. Templates are objects with the `TPL` prefix, registered in the [data model](../design/data-model.md).
- CI renders every default template, checks it with veraPDF, and compares it with a reference image to catch layout changes.
- Before the first paying business, the renderer is a new deployable ([ADR 0026](0026-modular-monolith.md)). It adds about A$5 a month at launch and about A$25 a month before paying businesses, taking those totals to about A$345 and A$500 ([ADR 0030](0030-hosting-aws.md)). The launch total comes close to the first budget alert at A$350. Chromium updates arrive through Dependabot with Playwright.
- The renderer's image is an exception to the chiseled base image ([ADR 0039](0039-security-baseline.md#supply-chain)).
- The renderer and the PDF text check need tests. The source editing design adds the reserved area for required content, the visibility check, and veraPDF on save, with tests including a template whose CSS tries to hide the required content.
- The template editor, when it is built, needs syntax errors shown on the line, a preview, version history and reset to default. Its design belongs to the settings design.
- A business that edits template source can produce an ugly document, but not an unsafe one, and never one missing required content.
