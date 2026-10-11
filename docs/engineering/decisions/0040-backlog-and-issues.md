# 0040. Backlog and issue tracking

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-11 |
| **Deciders** | Stuart Meeks |

## Context

Signboard needs somewhere to keep the work still to do: the order things will be built in, what is in progress, and bugs and requests from shops and contributors. Until now that has lived in the journal's "Next" sections and in Claude's own notes, which nobody else can see.

What to build is already written down. PRDs hold the user stories, with IDs and acceptance criteria, and design docs and ADRs hold the how and the why ([ADR 0011](0011-documentation-first.md)). A backlog tool that also held the detail would be a second source of truth, and the two would drift apart.

Signboard is built by one maintainer, part-time, with an AI that writes much of the code and documentation. Tools like Jira earn their place coordinating many people: workflows, permissions, sprints, estimates and reports for managers. Little of that applies here. AI also makes detailed tickets cheap to write, so a ticket's value is no longer in its detail, but in where it sits in the order and what state it is in.

The project works in the open ([ADR 0013](0013-work-in-the-open.md)), so the backlog should be public, and shops and contributors need a way to report bugs and ask for things.

## Decision

We will use **GitHub Issues**, with one **GitHub Project** board for the order and status of work.

### Specs in the docs, work in issues

- **PRDs, design docs and ADRs say what to build and why.** They are the only place a requirement is written down.
- **An issue is a unit of work that points to its spec**: the story IDs it delivers (such as `QUO-012`), or the design doc or ADR it implements. An issue never copies acceptance criteria or design detail. If an issue needs more explanation than a few lines, the spec is missing something, and the spec is fixed.
- Issues are created **when a phase starts**, not for the whole product in advance. Until then, the PRDs are the backlog.
- Bugs, chores and questions that belong to no story are issues in their own right.

### Issue types and labels

- **Types**, using GitHub's issue types: **Feature** (delivers stories), **Bug** (something does not work as its spec says) and **Task** (everything else, such as writing an ADR or upgrading a tool).
- **Labels** say where, not what:
    - one label per module, such as `module: sales` or `module: production`, matching the API's modules ([ADR 0020](0020-url-structure.md));
    - `documentation` and `ci` for work outside the modules;
    - `decision` for an ADR still to write or review;
    - `good first issue` and `help wanted` for contributors;
    - `accessibility`, for barriers affecting people with disabilities;
    - the workflow labels from other ADRs: `full-e2e` ([ADR 0028](0028-branching-ci-and-releases.md)) and `no-journal` ([ADR 0014](0014-build-journal.md)).
- GitHub's default `bug` and `enhancement` labels are removed, because issue types do that job.

### The board

One project, [**Signboard**](https://github.com/orgs/signboard-works/projects/1), owned by the `signboard-works` organisation and public.

| Field | Values |
|---|---|
| **Status** | Triage, Backlog, Ready, In progress, Done |
| **Phase** | Design, then the delivery phases in the [vision](../../vision.md): 1 Foundations, 2 Quotes, 3 Production, 4 Billing, 5 Migration |

- **Order is priority.** Items in Backlog and Ready are kept in the order they will be done, top first. There is no separate priority field and no estimates.
- **The maintainer sets the order.** Claude may create issues, link them to specs and pull requests, and move items through Triage, In progress and Done as work happens, but does not reorder the backlog or move work into Ready.
- New issues arrive in **Triage**. The maintainer accepts them into Backlog, or closes them with a reason.
- An item moves to **Done** when its pull request merges or the issue is closed.
- Views: the board by status, a backlog table in priority order, and a view by phase.

### Issues and pull requests

- A pull request that completes an issue says so in its description (`Fixes #12`), so merging it closes the issue. Pull request titles still follow Conventional Commits ([ADR 0028](0028-branching-ci-and-releases.md)), and every pull request still updates the journal ([ADR 0014](0014-build-journal.md)).
- **The journal records what happened; the board shows what is next.** Journal entries no longer keep their own list of next steps.

### Reporting

- **Issue forms** for bug reports and feature requests ask for what matters and remind reporters never to include personal or business data, such as customer names or prices ([ADR 0013](0013-work-in-the-open.md)). A feature request starts from the problem, not the solution.
- **Security problems are never issues.** The new-issue page links to private vulnerability reporting ([`SECURITY.md`](https://github.com/signboard-works/signboard/blob/main/SECURITY.md), [ADR 0039](0039-security-baseline.md)).
- Blank issues are turned off for the public, so every report uses a form.

## Options considered

1. **GitHub Issues and a GitHub Project**: public, free, next to the code and pull requests, closed automatically by merges, readable and writable by Claude with the `gh` command line, and open to outside reporters. Limited reporting and workflow rules, which a project this size does not need. **Chosen.**
2. **Jira**: powerful workflows, reports, and familiar to the maintainer; outside the repository and private by default, built for teams with managers, and a second place for specs to drift into.
3. **Linear**: fast and pleasant; the same objections as Jira, with less familiarity.
4. **A backlog file in the repository**: the simplest for Claude, versioned and reviewed; outsiders cannot report anything, and every change of status becomes a pull request.

## Consequences

- The journal's "Next" sections give way to the board. Today's outstanding work, such as the ADRs still to write, moves to issues.
- Each phase starts by creating its issues from the PRD's stories, in the order the maintainer chooses.
- Contributors can find work through `good first issue` and `help wanted` without asking.
- If the project outgrows GitHub Projects, issues can be exported or synced to another tool. The specs do not move, because they were never in the tracker.
