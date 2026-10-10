# Open questions

Questions for the design partner shop that shape the requirements. Answers are recorded here, then folded into the relevant PRD.

| # | Question | Affects | Answer |
|---|---|---|---|
| 1 | Which pricing methods are used today: area, machine or cutting time, per unit with quantity breaks, labour or design hours, fixed price? | [Pricing engine](../design/pricing-engine.md), [Quotes](prd-quotes.md) | |
| 2 | Does the current system price well, or are prices often overridden by hand? Where? | Pricing engine | |
| 3 | Is design billed separately or built into the job price? | Quotes | |
| 4 | How many proof revisions are typical, and how do customers approve proofs today? | Jobs, Customer email | |
| 5 | Who does installations, and how are they scheduled? | Jobs and kanban | |
| 6 | Does a job ever split across machines or staff in a way the current board can't show? | Jobs and kanban | |
| 7 | Does the website cart take real orders? Should they flow into Signboard? | Scope | |
| 8 | Are deposits taken? What payment terms do account customers get? | Sales orders, Invoices | |
| 9 | Which accounting system is used? (Believed to be Xero.) | [Accounting integration](../design/accounting-integration.md) | |
| 10 | Roughly how many quotes and jobs per month? | Design sizing | |
| 11 | Which current features are used every day, and which are never touched? | Scope | |
| 12 | What can be exported from the current system (customers, products, quotes, jobs, invoices), and in what format? | [Migration](prd-migration.md) | |
| 13 | Which devices are used on the shop floor (desktop, tablet, wall screen)? | Jobs and kanban | |
