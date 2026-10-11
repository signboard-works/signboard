# 0037. Localisation

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |


## Context

Signboard starts with one design partner shop in Australia, in English. It is open source and meant for sign and print shops anywhere ([vision](../../vision.md)), so other countries and languages will come, and translations may well be contributed by the community.

Localisation is far cheaper to build in from the start than to retrofit. Hard-coded strings spread through every screen, email and PDF; layouts assume English word lengths; and data stored in one country's shape, such as a phone number typed as `0412 345 678` or an address as one block of text, cannot be reliably reformatted later. Temporary often becomes permanent.

Some of this is already decided:

- Money, tax, time zones and units are in [ADR 0036](0036-money-tax-and-time.md): one currency per business, tax rates as configuration, the business's time zone for display, and metric units inside Signboard.
- Error messages may be translated, because clients branch on stable codes, never on wording ([ADR 0029](0029-api-errors-idempotency-versioning.md)).
- Formats come from the business's settings, never hard-coded ([front-end design](../design/front-end.md)).

This ADR decides how text is written, whose language and formats apply, and how country-specific data is stored. It does not add a second language.

## Decision

### English only at launch, ready for more

- **Signboard launches in English only.** No other language is translated until someone needs it.
- **Every piece of interface text is externalised from day one**: no text is hard-coded in components, emails or PDFs. CI enforces it (see [Enforcement](#enforcement)).
- **The source language is Australian English** (`en-AU`), matching the design partner shop and the British spelling used throughout the docs: "colour", "organisation", "cancelled". Other English locales, such as `en-GB` and `en-NZ`, fall back to it. US English would be a translation like any other.

### Whose language and formats

| Setting | Belongs to | Used for | Default |
|---|---|---|---|
| **Country** | Business | Defaults for currency, tax rates, time zone and locale ([ADR 0036](0036-money-tax-and-time.md)), address and phone formats, paper size, and country rules for documents, such as the title "Tax invoice" in Australia | Chosen at sign-up |
| **Locale** | Business | Number, currency and date formats for everything shown about the business's data, on screen and on its documents | From the country, such as `en-AU` |
| **Interface language** | User | The language of the app's own text: menus, labels, messages | The browser's language, if Signboard has it; otherwise the source language |
| **Document language** | Customer | The language of quotes, invoices and emails sent to that customer | The business's language |

- **Formats follow the business, not the viewer.** Everyone working on a business's data sees the same dates and amounts as its documents show, just as times are shown in the business's time zone ([ADR 0036](0036-money-tax-and-time.md)).
- **Dates are never shown in an all-number format that depends on the locale**, such as `3/11/2026`, which is 3 November in Australia and 11 March in the United States. Dates are shown with the month as a word, such as `3 Nov 2026` or `Tue 3 Nov`, on screen, in emails and on documents. The month name follows the language, so a German interface shows `3. Nov. 2026`. The one numeric exception is ISO 8601, `2026-11-03`, which cannot be misread and may be used wherever a compact date is wanted, such as dense grids and exports. Date inputs accept typing in the locale's order and show the date they understood with the month as a word.
- **A user's interface language is theirs**, across every business they belong to, because users are global ([ADR 0010](0010-api-first.md)).
- **What a customer receives is in the customer's language**, chosen by the business, never the language of whoever pressed "send".

### Text written by businesses is not translated

Names, descriptions, terms and conditions, and email and document templates are written by the business ([ADR 0006](0006-configuration-over-customisation.md)) and shown exactly as entered. At launch each template has one version. Templates in several languages, chosen by the customer's document language, can be added later without changing anything else.

### The API returns data, not wording

- The API returns codes and values, never display labels: a status is `in_production`, not "In production" ([ADR 0029](0029-api-errors-idempotency-versioning.md#json-types)). Clients translate codes into labels from their own catalogue.
- The only wording the API produces is in error responses (`title`, `detail` and each field error's `message`). These follow the request's `Accept-Language` header, falling back to the source language, and the response carries `Content-Language`.
- Numbers and dates in the API are never formatted.

### Front end

- **i18next with react-i18next** (both MIT), the most widely used translation library for React, with its JSON catalogues in i18next's own format. Plurals use the standard `Intl.PluralRules` categories, so languages with several plural forms work without code changes.
- **One catalogue per module**, such as `sales` and `production`, with keys named by module and purpose, such as `sales.quote.send.confirm`. Catalogues load with their module, so the app does not download every string at once.
- **Formatting uses the browser's `Intl` APIs** with the business's locale: numbers, currency, dates, lists, relative times ("3 days ago"), and the names of countries and currencies, which come from `Intl.DisplayNames` rather than our own lists.
- **Layouts allow for longer text.** Labels are not fixed-width, and buttons and columns cope with text 30 to 40% longer than English.
- **Right-to-left ready, not right-to-left supported.** Styles use CSS logical properties (`margin-inline-start`, not `margin-left`), so Arabic or Hebrew could be added later with Mantine's right-to-left support. No right-to-left language is supported at launch.

### Back end

- Text the server produces, such as error messages, email wording around the business's templates, and PDF labels, comes from **.NET resource files** through `IStringLocalizer`, the standard .NET approach.
- **Emails and PDFs use the recipient's language and the business's locale**, never the culture of the request or the server.
- **The containers run with full ICU**, not .NET's invariant globalisation mode, which some small base images turn on and which silently formats everything as if in the United States. A start-up check fails if ICU is missing.

### Country-specific data

- **Phone numbers** are stored in international E.164 format, such as `+61412345678`, validated and formatted with **libphonenumber** (libphonenumber-csharp, Apache 2.0, on the server; libphonenumber-js, MIT, in the browser). They are shown in the national format of the business's country.
- **Addresses** are stored as structured fields: address lines, locality, region, postcode and an ISO 3166-1 country code. They are validated and laid out per country from Google's open address metadata (libaddressinput, Apache 2.0), so an address prints correctly on a quote or a label in any country.
- **Paper size** for PDFs comes from the country: A4 by default, US Letter for the United States and Canada.
- **Sorting and searching text** by locale is decided in the search ADR.

### Translations

- **No translation platform until there is a second language.** When one is needed, we use a hosted platform with a free plan for open-source projects, such as Hosted Weblate or Crowdin, so contributors can translate without touching code. Translations arrive as pull requests and are reviewed like code.
- **Machine translation is never shipped unreviewed.** It may be used to draft a translation that a fluent speaker then reviews.
- A language is offered to users only when its catalogue is complete. Missing keys fall back to the source language.

### Enforcement

- **ESLint rejects literal text in components** (eslint-plugin-i18next, ISC), with a small allow-list for things that are not words, such as punctuation and units.
- **A pseudo-locale** turns every string into accented, bracketed and lengthened text, such as `[Šéñð qûöţé ~~~~]`. It is available in preview environments ([ADR 0028](0028-branching-ci-and-releases.md)), and one Playwright smoke run uses it, so hard-coded text and layouts that break with longer words are caught before merge ([ADR 0012](0012-testing.md)).
- **CI checks the catalogues**: every key used in code exists in the source catalogue, unused keys are reported, and a shipped language with missing keys fails the build.

## Options considered

### Front-end library

1. **i18next with react-i18next**: by far the most used (about 17 million weekly downloads of react-i18next, against about 3 million for react-intl), MIT, supported by every translation platform, with plurals and namespaces built in. **Chosen.**
2. **FormatJS (react-intl)**, BSD-3-Clause: uses the ICU MessageFormat standard natively, which handles plurals and selections inside one message; a smaller ecosystem, and ICU syntax is harder for casual translators.
3. **Lingui**, MIT: compiles messages at build time and is small; much less used.

### When to localise

1. **Externalise every string from day one, translate later**: costs a little discipline per screen; adding a language later is translation work, not a rewrite. **Chosen.**
2. **English hard-coded until a second language is needed**: faster at first; retrofitting every screen, email and PDF later is slow and misses strings.
3. **Translate into several languages at launch**: no demand yet, and every change would need translating before release.

### Whose formats

1. **The business's locale for all of its data, with dates never in a locale-dependent numeric format**: everyone sees the same as the documents, and nobody can misread a date's day and month whatever their own country. **Chosen**, by Stuart, with ISO 8601 (`2026-11-03`) allowed as the one numeric format.
2. **Each viewer's own locale**: familiar formats for everyone; screens and documents can disagree, and a quote's dates would look different to staff and to the customer reading the PDF.
3. **The business's locale for staff, each customer's document locale in the customer portal and on what they receive**: familiar for customers; two sets of formats for one business's data.

## Consequences

- Every screen, email and PDF is built with translation keys from the first one. Reviews check for hard-coded text, and ESLint and the pseudo-locale catch what reviews miss.
- Businesses gain a country and a locale, users an interface language, and customers a document language. Sign-up asks for the country and suggests the rest.
- Phone numbers and addresses are structured from the start, which the migration from the design partner shop's current system must map into ([migration PRD](../requirements/prd-migration.md)).
- The design for each module lists its catalogue, and its PDFs and emails use the recipient's language.
- Adding a language later needs a translation platform, a reviewer who speaks the language, and its fallback rules, but no code changes. A right-to-left language also needs a review of the layouts.

