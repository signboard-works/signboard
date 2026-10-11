# How we work

Signboard is built part-time, mostly with AI assistance, in the open. These rules keep it honest. Each is set out in full in its ADR.

- **Documentation first** ([ADR 0011](decisions/0011-documentation-first.md)). Plan first, then build. No code without an approved PRD story and design document. If the code proves a document wrong, the document is fixed in the same pull request.
- **Testing** ([ADR 0012](decisions/0012-testing.md)). Every acceptance criterion has an automated test that names its story ID. Every bug fix starts with a failing test. Code pull requests need 95% line coverage, and tests run against real PostgreSQL.
- **Work in the open** ([ADR 0013](decisions/0013-work-in-the-open.md)). Nothing about the project is secret, and we are honest about bugs and mistakes. The exceptions are security issues, which stay private until fixed, and personal or business data, which never appears.
- **Build journal** ([ADR 0014](decisions/0014-build-journal.md)). Every pull request that changes the repository updates the [journal](../journal/index.md), including what the AI got wrong or was overruled on. CI enforces it.
- **Branching, CI and releases** ([ADR 0028](decisions/0028-branching-ci-and-releases.md)). Trunk-based development: short-lived branches, squash-merged into `main` through pull requests whose titles follow Conventional Commits. Every pull request passes the required checks (including a Playwright smoke tier) and gets a preview environment. The full suite runs on `main`, and releases are semantic versions cut by release-please.
- **Coding with AI** ([ADR 0041](decisions/0041-coding-with-ai.md)). Whoever opens a pull request is accountable for every line, however it was written. Every change follows the [`contribute` skill](https://github.com/signboard-works/signboard/blob/main/.claude/skills/contribute/SKILL.md), from the board to a merged pull request, and every pull request gets an automated review from CodeRabbit. High-risk changes also get an adversarial review, and the maintainer reads every pull request that changes code. Instructions for any AI agent are in [`AGENTS.md`](https://github.com/signboard-works/signboard/blob/main/AGENTS.md).
- **Backlog** ([ADR 0040](decisions/0040-backlog-and-issues.md)). The order and status of work is on the public [project board](https://github.com/orgs/signboard-works/projects/1), as GitHub issues that point to their stories in the PRDs. Specs live only in the docs. Report bugs and ideas with the [issue forms](https://github.com/signboard-works/signboard/issues/new/choose); report security problems privately ([`SECURITY.md`](https://github.com/signboard-works/signboard/blob/main/SECURITY.md)).

## Templates

New documents start from a template in [`docs/engineering/templates/`](https://github.com/signboard-works/signboard/tree/main/docs/engineering/templates):

- [PRD template](https://github.com/signboard-works/signboard/blob/main/docs/engineering/templates/prd-template.md)
- [Design document template](https://github.com/signboard-works/signboard/blob/main/docs/engineering/templates/design-doc-template.md)
- [ADR template](https://github.com/signboard-works/signboard/blob/main/docs/engineering/templates/adr-template.md)
- [Journal template](https://github.com/signboard-works/signboard/blob/main/docs/engineering/templates/journal-template.md)
