# Personas

`init-personas.sh` bootstraps a file-based, issue-driven workflow for AI-assisted
software development. It creates shared session instructions, project memory,
specification tracking, work directories, persona prompts, and dispatcher scripts
that select exactly one persona for each session.

The workflow is designed around `bd` issues, `specs.md` as the current source of
intent, `changes/` files as in-flight capability specs, `specs/` files as archived
capability history, and `STATE.md` as short shared memory.

## Quick start

From the root of the project you want to initialize:

```bash
./init-personas.sh
```

Then provide a project-level `specs.md` file and start your agent with:

```text
read instructions.md and follow it
```

The generated `instructions.md` tells the agent to prime `bd`, read project state,
check stale claims, run `.personas/load-context.sh`, then read only the persona file
selected for that session.

## What gets created

The script creates these files and directories if they do not already exist:

```text
project/
├── instructions.md
├── STATE.md
├── specs-inventory.md
├── changes/
│   └── .gitkeep
├── specs/
│   └── .gitkeep
├── codebase/
│   └── .gitkeep
└── .personas/
    ├── architect.md
    ├── developer.md
    ├── documentation.md
    ├── gap-analyst.md
    ├── investigator.md
    ├── load-context.sh
    ├── mapper.md
    ├── monitor.md
    ├── refiner.md
    ├── resolver.md
    ├── reviewer.md
    ├── security.md
    ├── select-issue.sh
    ├── steward.md
    └── tester.md
```

It also ensures `changes/`, `specs/`, and `codebase/` exist with `.gitkeep` files,
and marks `.personas/load-context.sh` and `.personas/select-issue.sh` executable.

The script is safe to re-run. Existing files are skipped rather than overwritten,
while the core directories and script execute bits are re-ensured.

## Generated files

`instructions.md` is the entry point for every agent session. It defines the
onboarding sequence, stale in-progress claim cleanup, context-loading step, and
commit message convention.

`STATE.md` is the short working memory for the project. Every persona reads it at
session start and appends a concise session log entry at the end. It contains
sections for blockers, architectural decisions, capability status, known concerns,
and the session log.

`specs-inventory.md` is owned by the Steward. It starts as a placeholder and is
populated from `specs.md` on the first Steward session. Each inventory entry is a
discrete, independently verifiable requirement with coverage status.

`changes/` holds in-flight capability change files. These are the shared working
specifications for planning, implementation, test, refinement, security review, and
final review.

`specs/` holds archived change files after the Reviewer decides a capability is
ready to become historical specification.

`codebase/` holds long-lived structural documentation maintained by the Mapper,
such as stack, architecture, conventions, concerns, security notes, health history,
and changelog-style summaries.

`.personas/select-issue.sh` selects the persona and issue for a session. It outputs
a small JSON object containing a persona path and either a full issue object or
`null`.

`.personas/load-context.sh` wraps selection into a readable session context. It
prints the issue to inspect, files to read, and final persona file to load.

## Session flow

The generated `instructions.md` runs sessions in this order:

1. Run `bd prime`.
2. Read `STATE.md` fully.
3. Run `git status` and `git log --oneline -5`.
4. If there are uncommitted changes, commit or stash them before proceeding.
5. Check `bd list --status in_progress --json` for stale claims.
6. For each in-progress issue, inspect the most recent commit mentioning its ID.
7. If the claim is older than 24 hours and no relevant uncommitted work exists,
   unclaim it with `bd update <id> --status open --assignee ""`.
8. Run `.personas/load-context.sh`.
9. If context loading exits with code `1`, stop; it has written a note explaining
   the malformed issue.
10. If context loading exits with code `0`, read the printed context fully.
11. Read only the persona file named in the final `PERSONA` block.
12. Follow that persona until it tells the agent to stop.

## Dispatch order

Dispatch is handled by `.personas/select-issue.sh`.

Longitudinal personas run before the issue queue is consulted. They receive
`issue: null` and do not claim or close issues.

| Order | Persona | Trigger |
|---|---|---|
| 1 | Monitor | Commits since last `monitor:` commit reach `PERSONA_MONITOR_INTERVAL` |
| 2 | Steward | Commits since last `steward:` commit reach `PERSONA_STEWARD_INTERVAL`, or `specs.md` is newer than `specs-inventory.md` |
| 3 | Mapper | Commits since last `map:` commit reach `PERSONA_MAPPER_INTERVAL` |

The interval defaults in `select-issue.sh` are `30` commits for Monitor, Steward,
and Mapper. They can be overridden with environment variables:

```bash
PERSONA_MONITOR_INTERVAL=10 \
PERSONA_STEWARD_INTERVAL=5 \
PERSONA_MAPPER_INTERVAL=20 \
.personas/select-issue.sh
```

If no longitudinal persona is due, `select-issue.sh` reads:

```bash
bd ready -n 100 --json
```

The first matching ready issue wins:

| Order | Persona | Trigger |
|---|---|---|
| 1 | Resolver | Ready `task` labeled `ambiguity` |
| 2 | Architect | Ready `task` labeled `plan` |
| 3 | Security | Ready `task` labeled `security` |
| 4 | Reviewer | Ready `task` labeled `review` |
| 5 | Tester | Ready `task` labeled `test` |
| 6 | Investigator | Ready `bug` whose description does not contain `root-cause:` |
| 7 | Developer | Ready `feature`, ready `bug` with `root-cause:`, or ready untagged `task` |
| 8 | Refiner | Ready `task` labeled `refine` |
| 9 | Documentation | Ready `task` labeled `docs` |
| Fallback | Gap Analyst | Queue is empty or no ready issue matches |

## Context loading

`.personas/load-context.sh` prints a complete session context:

- selected persona
- selected issue ID, type, and labels, if any
- commands the agent must run, usually `bd show <id> --json`
- files the agent must read
- final persona file to load

For `feature`, `bug`, and untagged `task` issues, the issue description must include
a change file reference:

```text
Change file: changes/<slug>.md
```

If that reference is missing, `load-context.sh` writes a note to the issue, prints a
`STOP -- MALFORMED ISSUE` message, and exits with code `1`.

If a referenced change file exists, it is listed as required context. If the file is
referenced but missing, the persona is expected to create it as its first action
before doing other work. For issue types that do not require a change file, `specs.md`
is used as fallback context when present.

## Persona responsibilities

| Persona | Responsibility |
|---|---|
| Developer | Implement selected features, rooted bugs, or untagged tasks; write required unit tests; create downstream refine, test, and optional docs issues |
| Architect | Turn a change file into a concrete technical design, verification commands, and decomposed implementation issues |
| Tester | Verify correctness with integration or end-to-end tests; report bugs rather than fixing implementation |
| Refiner | Improve existing work without adding new features; close quality gaps and file remaining findings |
| Reviewer | Judge whether accumulated work matches the change file; file findings or archive `changes/<slug>.md` to `specs/<slug>.md` |
| Documentation | Update documentation based on actual implementation only |
| Resolver | Resolve blocking ambiguity issues quickly and return the workflow to progress |
| Steward | Own `specs-inventory.md`; sync it to `specs.md`, classify coverage, and update capability status in `STATE.md` |
| Gap Analyst | Run when the ready queue is empty; find uncovered or partial requirements and create new change files and plan issues |
| Security | Audit for vulnerabilities and file security findings rather than fixing them |
| Investigator | Diagnose unrooted bugs and create actionable rooted bug issues |
| Monitor | Assess workflow health across the project and file drift, stall, hotspot, or test-ratio issues |
| Mapper | Keep `codebase/` accurate and current from verified structural observations |

## Change-file lifecycle

Capability work is organized around `changes/<slug>.md`.

A normal change file includes:

- `## Why`
- `## Covers`
- `## Preferences`
- `## Scope`
- `## Out of scope`
- `## Constraints`
- `## Decision log`
- `## Open questions`
- `## Design`
- `## Verification commands`
- `## As built`

Typical lifecycle:

1. The Gap Analyst identifies a gap in `specs-inventory.md` or project history.
2. The Gap Analyst writes `changes/<slug>.md` and creates one `plan` task.
3. The Architect fills in design and verification commands, then creates
   implementation issues.
4. The Developer implements scoped issues and unit tests.
5. The Tester verifies behavior and files bugs for failures.
6. The Refiner performs focused quality improvements.
7. The Security persona audits when a security task exists.
8. The Reviewer accepts or rejects the completed capability.
9. On acceptance, the Reviewer moves the file from `changes/` to `specs/` and fills
   `## As built`.
10. The Steward later updates `specs-inventory.md` and `STATE.md` based on the new
   coverage picture.

## Specification inventory

`specs.md` is the current statement of intent. `specs-inventory.md` is the
Steward-maintained decomposition of that intent into traceable requirements.

Inventory entries are expected to look like:

```markdown
## SPEC-001

**Section**: Authentication > Token refresh
**Quote**: "<verbatim requirement>"
**Type**: functional
**Keywords**: token, refresh, expiry, session, rotation
**Coverage**: UNCOVERED
**Verified**: never
**Last reviewed**: 2026-04-27
```

Coverage values used by the generated Steward include:

- `UNCOVERED`
- `PARTIAL: <slug>`
- `COVERED: <slug>`
- `CONFLICTED: <slug-1>/<slug-2>`
- `SUPERSEDED: <reason>`

The Steward writes a Traceability Table and Review Readiness Summary into
`STATE.md` under `## Capability status`. The Gap Analyst consumes that summary when
deciding whether to create more work or declare the project complete.

## Commit conventions

Most issue-bound personas use this commit message format:

```text
<type>(<scope>): <short description>
```

Valid types are:

- `feat`
- `fix`
- `refine`
- `test`
- `review`
- `docs`
- `security`
- `investigate`
- `plan`
- `map`
- `monitor`
- `chore`

The scope is usually the change-file slug. For sessions not tied to a capability,
the generated personas use fixed scopes such as `resolver` or `gap-analyst`.

Longitudinal personas use fixed subject prefixes because `select-issue.sh` uses
commit history to decide when they are due:

- `monitor: <summary>`
- `steward: <summary>`
- `map: <summary>`

Examples:

```text
feat(auth): implement JWT refresh token rotation
plan(auth): design token refresh architecture, create 3 issues
review(auth): archive change file, file 2 findings
chore(gap-analyst): identify 2 gaps, create auth and billing
steward: sync inventory: 3 new, 1 superseded, 5 verified
monitor: health check -- test ratio 18%, 1 hotspot flagged
map: update codebase conventions and architecture
```

## Prerequisites

The generated workflow assumes these tools are available:

- `bash`
- `git`
- `jq`
- `beads` (`bd`)
- an agent that can read files and run shell commands

The init script itself is plain Bash and does not install dependencies.
