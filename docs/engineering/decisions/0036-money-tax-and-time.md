# 0036. Money, tax, time and quantities

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

## Context

Almost everything Signboard does involves money, tax, dates or measurements. A quote line is priced by area or machine time ([pricing engine](../design/pricing-engine.md)), carries GST, totals into a document, becomes an invoice and syncs to an accounting system ([ADR 0005](0005-accounting-as-plugin.md)) that will check our totals to the cent. Due dates decide what is overdue, and reminders run at a time of day in the business's time zone ([ADR 0034](0034-background-work.md)).

These are classic sources of bugs:

- **Floating-point arithmetic.** `0.1 + 0.2` is not `0.3` in binary floating point, and totals drift by cents.
- **Rounding.** .NET's `Math.Round` rounds halves to the nearest even number by default (banker's rounding), so `2.345` becomes `2.34`, where the tax office and accounting systems expect `2.35`. Rounding per line or per document gives different totals.
- **Tax that changes.** If a tax rate changes, an old invoice recalculated with the new rate no longer matches what was sent.
- **Dates mistaken for moments.** A due date of 1 November stored as midnight UTC becomes 31 October in Sydney. `DateTime` mixes local and UTC values in one type.
- **Units.** A quantity of `2.5` means nothing without knowing whether it is square metres, metres or hours.

Each needs one rule, decided before the first price is calculated. The JSON types the API uses for these values are decided in [ADR 0029](0029-api-errors-idempotency-versioning.md#json-types).

## Decision

### Money

- **Never floating point.** Amounts are `numeric` in PostgreSQL and `decimal` in .NET. `float` and `double` are never used for money, rates or quantities, which an architecture test checks ([ADR 0012](0012-testing.md)).
- **In the API, amounts are plain JSON numbers**, such as `"total": 1234.5` ([ADR 0029](0029-api-errors-idempotency-versioning.md#json-types)). Integrators do not need to look up whether a numeric field is a string.
- **Precision.**
    - Unit rates and quantities keep up to **4 decimal places**, because sign trade rates often need them, such as A$42.375 per square metre.
    - Line amounts, tax and document totals are **rounded to the currency's minor unit** (cents for the Australian dollar, from ISO 4217).
    - Columns are `numeric(15, 4)`: at most 15 significant digits, up to 99,999,999,999.9999. Every value therefore fits exactly in a JavaScript number, so no value changes on its way through a JSON client.
- **Commercial rounding is separate.** Rounding a price up to the nearest $5, or a minimum charge, is pricing configuration in the [pricing engine](../design/pricing-engine.md). The rules here are only about arithmetic.

### Currency

- **One currency per business**, an ISO 4217 code such as `AUD`, chosen when the business is created. It can be changed until the business issues its first document, then it is locked.
- **Each quote, sales order and invoice carries its own `currency` field**, which applies to every amount on it. Amounts are plain numbers, not `{amount, currency}` objects. Documents in more than one currency can therefore be added later without breaking the API.
- **Multi-currency is out of scope at launch**, and so are exchange rates.
- Connecting an accounting system checks that its base currency matches the business's currency.

### Rounding

- **Half away from zero**: `2.345` becomes `2.35` and `-2.345` becomes `-2.35`. This is what the Australian Taxation Office and Xero expect.
- All rounding goes through Signboard's own small `Money` type (an amount and a currency) and its rounding helper. A banned-API analyser rejects `Math.Round` and `decimal.Round` without an explicit `MidpointRounding`, so banker's rounding cannot slip in.
- We do not use a money library such as NodaMoney. Our needs are an amount, a currency, its minor unit and one rounding rule, which is a small type of our own.

### Tax

- **Tax rates are configuration** ([ADR 0006](0006-configuration-over-customisation.md)). Each business has its own list, such as GST 10%, GST-free and export, each with a name, a percentage and the matching tax code in its accounting system. A new business starts with its country's standard set. Their API is designed with invoices.
- **Each line has one tax rate.**
- **Tax is calculated per line and rounded per line.** Document totals are the sums of the rounded lines. This is how Xero calculates tax, so the totals we sync match the accounting system's to the cent. We send line amounts and tax amounts explicitly, never leaving the accounting system to recalculate them.
- **Prices are tax-exclusive by default**, which suits trade customers. A business setting makes them tax-inclusive, such as for walk-in retail customers. For tax-inclusive lines, tax is worked out from the gross amount (`gross × rate ÷ (100 + rate)`, rounded) and the net amount is the remainder, so the gross amount the customer saw is never changed by rounding.
- **Rates are fixed once a document is issued.** Each line stores the tax rate's name, code and percentage. Drafts follow the current rates; once a quote is sent or an invoice issued, its lines keep the rates they were issued with and are never recalculated. A change to the GST rate never alters an old invoice.
- **Countries.** The model fits countries with a single GST or VAT rate per line, starting with Australia. New Zealand (15%) and most VAT countries need only configuration. US sales tax and compound taxes, such as Canada's GST plus PST, are out of scope at launch, but the model does not rule them out.

### Time

| Kind | Examples | PostgreSQL | .NET (NodaTime) | API |
|---|---|---|---|---|
| **A moment in time** | When something was created, sent or approved | `timestamptz` | `Instant` | RFC 3339 in UTC, `2026-10-11T03:15:00Z`. Names end in `At`: `createdAt`, `sentAt` |
| **A calendar date** | Due dates, invoice dates, install dates | `date` | `LocalDate` | `2026-10-11`. Names end in `Date`: `dueDate`, `invoiceDate` |
| **A time of day** | A reminder at 9 am | `time` | `LocalTime` | `09:00` |
| **A future appointment** | An installation at 10 am next Tuesday | `date` and `time`, in the business's time zone | `LocalDateTime` | A date and a time of day, such as `"installDate": "2026-10-20", "installTime": "10:00"`, in the business's time zone |

- **Moments are stored in UTC** and returned in UTC. The API accepts any RFC 3339 offset on input and converts it. A time with no offset is refused with `422`.
- **A calendar date has no time and no zone.** It is the same date for everyone. Questions such as "overdue" or "due today" are answered in the business's time zone.
- **Future appointments keep their local time.** An installation at 10 am stays at 10 am even if daylight-saving rules change before it happens. It is converted to a moment only when needed, such as to schedule a reminder.
- **Each business has a time zone**, an IANA name such as `Australia/Sydney`, set when the business is created from the browser's zone and changeable in its settings.
- **Times are shown in the business's time zone**, the same as on its documents, for staff and customer users. When the viewer's browser is in a different zone, times are labelled with the zone, such as "10:00 AEDT".
- **Recurring schedules a business would notice run in its time zone** ([ADR 0034](0034-background-work.md)). System schedules run in UTC.

### Clocks and libraries

- **NodaTime** (Apache 2.0) for dates and times, with the Npgsql NodaTime plugin for EF Core (PostgreSQL licence) and NodaTime's System.Text.Json serialisation. Its separate types, `Instant`, `LocalDate`, `LocalTime` and `LocalDateTime`, make it impossible to pass a date where a moment is needed.
- **Time comes from an injected `IClock`.** A banned-API analyser rejects `DateTime.Now`, `DateTime.UtcNow`, `DateTimeOffset.Now` and `DateTimeOffset.UtcNow`. Tests use a fake clock, so anything time-based can be tested at any moment, including across daylight-saving changes.
- The time zone database comes with NodaTime and is kept up to date by Dependabot ([ADR 0028](0028-branching-ci-and-releases.md)).

### Quantities

- **Every quantity has a unit**, from a fixed list in code: `each`, `mm`, `m`, `m2` and `hour` to start with. In the API a quantity sits next to its unit: `"quantity": 2.5, "unit": "m2"`.
- **Metric inside Signboard.** Sizes are millimetres, lengths metres, areas square metres. Machine and labour time are **decimal hours**, such as `1.5`, never minutes or `1:30`. Field names say the unit where it is fixed, such as `widthMm` or `machineHours`.
- Showing other units, such as feet and inches, can come later as display configuration. What is stored stays metric.

### Formatting

Numbers, amounts and dates are formatted for display from the business's locale, using the browser's `Intl` formatting, never hard-coded ([front-end design](../design/front-end.md)). Dates are shown with the month as a word, such as `3 Nov 2026`, or as ISO 8601 (`2026-11-03`), never in a locale-dependent numeric format such as `3/11/2026` ([ADR 0037](0037-localisation.md)). Formatting is for display only: the values in the API are never formatted.

### Arithmetic outside the API

- **The API calculates every price, tax amount and total.** The web app displays them. Where the web app calculates something on screen, such as a live preview while a quote line is edited, it uses a decimal library, chosen in the front-end design, and the API's result is what is saved.
- The generated .NET client reads amounts into `decimal`. The API reference tells integrators to do the same in their language, and never to add up amounts in floating point.

## Options considered

### Amounts in the API

1. **JSON numbers**: what integrators expect, and what Xero and MYOB use. Values are limited to 15 significant digits so they survive JavaScript parsing exactly. **Chosen**, by Stuart: "numbers are numbers", so nobody has to look up a numeric field's type.
2. **Decimal strings**, as PayPal and Shopify use: keep the scale (`"1234.50"`) and push clients towards decimal types; every integrator must convert before use, and numbers would be strings in some fields and not others.
3. **Integer minor units**, as Stripe uses: no fractions at all; cannot hold rates with more than two decimal places without a second convention.

### Tax rounding

1. **Per line**: matches Xero, and each line's tax can be shown and explained. **Chosen.**
2. **Per document**: can differ from the sum of the lines by a cent; would not match what we sync.

### Dates and times in .NET

1. **NodaTime**: separate types for moments, dates and local times; first-class support in Npgsql; a clock interface for tests. **Chosen.**
2. **The .NET built-ins** (`DateTimeOffset`, `DateOnly`, `TimeOnly`, `TimeProvider`): no dependency, and much better than they were; `DateTime` and its `Kind` remain one mistake away, and time zone rules are weaker.

### Display time zone

1. **The business's time zone, labelled when the viewer is elsewhere**: everyone at a business sees the same times as its documents show. **Chosen.**
2. **The viewer's browser zone**: natural for one person; staff and customers in different zones would see different times for the same appointment.

## Consequences

- The `Money` type, the rounding helper, the tax calculation and the unit list live in a shared library used by every module. They are tested with property-based tests for rounding and tax ([ADR 0012](0012-testing.md)), including tax-inclusive lines and negative amounts such as credits.
- Every line on a quote, order or invoice stores its tax rate's name, code and percentage. The invoices and accounting designs map tax rates to each accounting system's tax codes.
- The architecture tests and banned-API analysers check for floating-point money, unqualified rounding and the system clock.
- Validation refuses values with too many decimal places or digits, with `422 too_many_decimal_places` ([ADR 0029](0029-api-errors-idempotency-versioning.md#json-types)).
- Each business needs a currency, a time zone and a set of tax rates when it is created. Sign-up asks for the country and suggests them.
- Supporting a country with compound or US-style sales tax would need a new ADR.
