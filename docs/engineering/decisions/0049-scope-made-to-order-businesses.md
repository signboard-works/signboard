# 0049. Scope: built for sign and print, open to any made-to-order business

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-12 |
| **Deciders** | Stuart Meeks |

## Context

Signboard is built for small sign and print shops, with one design partner shop ([vision](../../vision.md)). Most of what it does is not specific to signs: a quote becomes a sales order, the sales order becomes jobs that move across a production board, and the work is invoiced. Embroiderers, engravers, joiners, metal fabricators, glaziers and 3D-printing bureaus work the same way. The pricing methods in the [pricing engine](../design/pricing-engine.md) (area, length, machine time, per unit with quantity breaks, labour hours and fixed prices, combined on one line) price their work as well.

There is no other trade in view. The aim is to keep the option open, not to build for one. Names are cheap to change while Signboard is still being designed, and expensive once there is real data, a public API ([ADR 0029](0029-api-errors-idempotency-versioning.md)) and integrations relying on them. The tenant is already a **business**, never a "shop", and everything that differs between businesses is already configuration ([ADR 0006](0006-configuration-over-customisation.md)).

The vision also warns against the opposite mistake: generic tools fail these businesses because they don't understand how the work is priced and produced. Signboard must not become one of them.

## Decision

### The model is trade-neutral

- Names in the data model, the API, events, permissions, object ID prefixes and the UI's own labels describe made-to-order work in general, not signs. For example, **material** rather than substrate, **work centre** rather than laminating or routing, **site visit** rather than installation, and **item** rather than sign.
- Sign and print terms are used where they belong: in the pitch, in examples and sample data, in configuration a business enters, such as its own product names and production stages, and in the sign and print starter pack.
- A new PRD, design doc or ADR uses trade-neutral names for anything that becomes part of the model. Reviews check for it, as they check for "business" rather than "shop".

### Trade-specific features are optional

- A feature that some made-to-order trades need and others don't, such as **proof approval**, **artwork files** or **installation and site visits**, is designed as a capability a business switches on in its settings ([ADR 0006](0006-configuration-over-customisation.md)). It is not an edition ([ADR 0046](0046-entitlements.md)) or a feature flag ([ADR 0033](0033-feature-flags.md)).
- The sign and print starter pack switches on what sign shops need, so they get the full product with no setup.

### Starter packs carry the trade

- A **starter pack** is configuration only: pricing templates, products, materials, work centres, document wording and the capabilities to switch on, such as proofs and site visits, for one trade, applied when a business is set up and freely changed afterwards.
- Signboard launches with one starter pack, **sign and print**, built with the design partner shop. Another trade would get its own pack, with no code change.

### The pitch and the scope stay specific

- Signboard is presented as **built for sign and print shops**, and **works for any made-to-order business**. The website, the vision and the documentation lead with sign and print.
- Requirements are still validated against the design partner shop. No feature is built for another trade until a real business in that trade asks for it, as the vision's non-goals already say.

## Options considered

1. **Trade-neutral model, sign and print pitch and starter pack**: keeps other trades possible at the cost of a naming pass now, without diluting the product for the businesses it is built for. **Chosen.**
2. **Sign and print only, in the model as well**: the most specific; renaming the model later would break the API, integrations and every business's data.
3. **A general job-management product for any trade**: the widest market; competes with the generic tools the vision says fail these businesses, and has no design partner to say what "good" means.

## Consequences

- The PRDs, design docs, glossary and other ADRs are reviewed once for sign-specific names in the model and corrected in the same pull request as this ADR.
- The glossary separates model terms, defined in general, from sign and print trade terms, kept as examples, and adds material, unit of measure, site visit, capability and starter pack.
- Pricing methods are general: a measured quantity in one of the units of measure, a collection operations users manage ([ADR 0036](0036-money-tax-and-time.md#quantities)), of which area and length are the sign and print cases, rather than a fixed list ([pricing engine](../design/pricing-engine.md)). A quote line has no fixed width and height.
- The built-in Installer role becomes **Field staff**, offered when a business uses site visits ([ADR 0022](0022-authorisation-and-roles.md)).
- Details of a customer's item, such as a vehicle for a wrap, are not fixed fields in the model. How a business records them is designed with jobs.
- The vision says "built for sign and print first; works for any made-to-order business".
- The setup design defines starter packs and how one is applied to a new business.
- The name Signboard stays. It suits the first market, and it can be revisited if another trade ever becomes a large part of the business.
