# Design

Technical design documents describe how each part of Signboard works: its data, API contract, behaviour and tests. A design document is approved before the code it describes is written, and is corrected in the same pull request if the code proves it wrong ([ADR 0011](../decisions/0011-documentation-first.md)).

| Document | Covers | Status |
|---|---|---|
| [Architecture](architecture.md) | The overall shape: modules, deployables and how they fit together | Draft |
| [Data model](data-model.md) | Core entities, tenancy, and the object ID prefix register | Draft |
| [Front end](front-end.md) | The single-page app, wrapper components, `DataGrid` and table views | Draft |
| [Pricing engine](pricing-engine.md) | How quote lines are priced | Draft |
| [Accounting integration](accounting-integration.md) | The accounting plugin interface and sync | Draft |
| [Email delivery](email-delivery.md) | The outbox, sending streams and delivery events | Draft |
| [Deployment](deployment.md) | Hosting, the edge router and self-hosting | Draft |
