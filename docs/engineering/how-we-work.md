# How we work

Signboard is built part-time, mostly with AI assistance, in the open. These rules keep it honest. Each is set out in full in its ADR.

- **Documentation first** ([ADR 0011](decisions/0011-documentation-first.md)). Plan first, then build. No code without an approved PRD story and design document. If the code proves a document wrong, the document is fixed in the same pull request.
- **Testing** ([ADR 0012](decisions/0012-testing.md)). Every acceptance criterion has an automated test that names its story ID. Every bug fix starts with a failing test. Code pull requests need 95% line coverage, and tests run against real PostgreSQL.
- **Work in the open** ([ADR 0013](decisions/0013-work-in-the-open.md)). Nothing about the project is secret, and we are honest about bugs and mistakes. The exceptions are security issues, which stay private until fixed, and personal or business data, which never appears.
- **Build journal** ([ADR 0014](decisions/0014-build-journal.md)). Every pull request that changes the repository updates the [journal](../journal/index.md), including what the AI got wrong or was overruled on. CI enforces it.

## Templates

New documents start from a template in [`docs/engineering/templates/`](https://github.com/signboard-works/signboard/tree/main/docs/engineering/templates):

- [PRD template](https://github.com/signboard-works/signboard/blob/main/docs/engineering/templates/prd-template.md)
- [Design document template](https://github.com/signboard-works/signboard/blob/main/docs/engineering/templates/design-doc-template.md)
- [ADR template](https://github.com/signboard-works/signboard/blob/main/docs/engineering/templates/adr-template.md)
- [Journal template](https://github.com/signboard-works/signboard/blob/main/docs/engineering/templates/journal-template.md)
