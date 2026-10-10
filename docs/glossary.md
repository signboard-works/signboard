# Glossary

Trade terms and Signboard's own vocabulary. If a word appears in a PRD or design doc and isn't obvious to someone outside the sign industry, it belongs here.

## Signboard concepts

| Term | Meaning |
|---|---|
| **Quote** | A priced offer to a customer. Quotes are versioned; a sent quote is never edited, only revised. A quote keeps one ID, and its versions are numbered beneath it. |
| **Sales order** | An accepted quote, locked as the agreed scope and price. Deposits and invoices hang off it. |
| **Job** | A unit of production work created from a sales order. One sales order can produce several jobs. |
| **Work centre** | A stage or station in production, such as design, print, laminate, cut, fabricate or install. |
| **Kanban board** | The live view of jobs by work centre or stage. A view of jobs, not a separate record. |
| **Invoice** | A bill raised from a sales order: full, deposit or progress. Synced to the accounting system. |
| **Pricing method** | How a quote line component is priced: by area, length, time, unit, or a fixed amount. |
| **Proof** | Artwork sent to the customer for approval before production. |
| **Business** | A sign or print shop using Signboard, with its staff as members of its business account. Signboard says "business" rather than "shop" in its model, because "shop" suggests an online store. In the trade sense ("sign shop", "shop floor"), "shop" is still used. |
| **Tenant** | One business using the hosted Signboard service, together with its customers. Every business's data is isolated from every other business's. |
| **User** | A person who can sign in, identified by a unique email address. Users are global: a user belongs to no single business. |
| **Account** | Something users can be members of. There are three kinds: a business account (the business and its staff), a customer account (one of a business's customers) and the operations account (the people who run the hosted service). |
| **Operations account** | The platform's own account. Membership gives no access to any other account; operations users add themselves to another account when they need to; the account's administrators are notified, and it is audited. |
| **Membership** | A user's link to an account, with a role. A user can have many, of any kind. |
| **Customer** | An organisation or person that buys from a business: a customer account. Each customer belongs to one business, so an organisation that buys from two businesses is two customers. Its users sign in to that business's customer portal. |
| **Customer portal** | The part of Signboard where customer users view and approve quotes and proofs, and see their orders and invoices. Each business has its own, at the business's subdomain. It is the same app and the same URLs as the staff app, showing what a customer user may see, and uses the same API as everything else. |
| **Actor** | Whoever is making an API request: a user acting in one membership, or an integration client belonging to a business or customer account. The actor decides the tenant and which records and fields are visible. |
| **Object ID** | The one identifier every object in Signboard has, such as `JOB-4821-0937-1156`: a type prefix and random digits in groups of four, with the length fixed per type. It is also the database primary key. |
| **Identity service** | Signboard's central sign-in service at `id.signboard.works`. Every business's host signs users in through it, so one sign-in works across businesses. |
| **Passkey** | A sign-in credential stored on a device or password manager, used instead of a password. It cannot be phished and counts as multi-factor on its own. |
| **API token** | A secret an API client uses to call the API. It belongs to one account, has fixed permissions and an expiry date, and is stored only as a hash. |
| **Audit trail** | The record of every change to every object, with who made it and a snapshot of the object afterwards, plus security events. |
| **Soft deletion** | Marking a row as deleted instead of removing it. The default for every object in Signboard. |
| **RQL** | Resource Query Language: the query-string syntax Signboard's API uses for filtering, sorting, selecting fields and paging, for example `and(eq(status,in_production),ge(dueDate,2026-11-01))`. |
| **Table view** | A user's saved arrangement of a grid: visible columns, order, widths, pinned columns, sort, filters, page size and density. Belongs to the user and is stored in their browser. |
| **Default view** | A business's starting table view for a grid, set by a business administrator and used by anyone without a table view of their own. |
| **Field visibility** | The rule that decides which fields of a record an actor can see. Hidden fields are left out of API responses entirely. |

## Trade terms

| Term | Meaning |
|---|---|
| **ACM** | Aluminium composite material: two thin aluminium sheets bonded to a plastic core. A common rigid sign substrate. |
| **Substrate** | The material a graphic is printed on or applied to (ACM, acrylic, corflute, foamboard, vinyl, fabric). |
| **Corflute** | Corrugated plastic sheet, used for cheap temporary signs. |
| **Vinyl** | Self-adhesive film, printed or cut, used for window graphics, lettering and wraps. |
| **Laminate** | A clear protective film applied over a print to protect it from UV and abrasion. |
| **Large format** | Printing wider than standard paper sizes, typically on roll-fed or flatbed printers. |
| **Bleed** | Extra artwork beyond the final cut line so no unprinted edge shows after trimming. |
| **Print and cut** | Printing a graphic, then contour-cutting around its shape on a cutter. |
| **Contour cut** | Cutting along the outline of a printed shape rather than a straight edge. |
| **CNC routing** | Computer-controlled cutting of rigid materials such as ACM, acrylic and timber. |
| **Laser cutting** | Cutting or engraving with a laser; common for acrylic lettering and engraved items. |
| **Flatbed cutter** | A cutting table with knife tools for board materials and packaging. |
| **Wrap** | Printed vinyl applied over a vehicle or object, partially or fully. |
| **Illuminated sign** | A sign with internal or external lighting (LED modules, lightboxes). |
| **Pylon sign** | A tall, freestanding sign, often at a property entrance or roadside. |
| **Roundel** | A circular sign or logo panel. |
| **Install** | Fitting a finished sign or wrap on site, often with travel, access equipment and permits. |
