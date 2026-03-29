# Personas

A workflow automation system for AI-assisted software development using specialized agent personas.

## Overview

Personas implements a structured development workflow where different AI agents take on specific roles (Developer, Tester, Refiner, Reviewer, Documentation Specialist, Analyst) to iteratively build software from specifications. Each persona has a focused responsibility and follows a defined protocol, ensuring systematic progress through the development lifecycle.

## How It Works

1. **Specifications-driven**: Work starts from a `specs.md` file that defines project requirements
2. **Issue tracking**: Uses `bd` (backlog daemon) to manage issues with types (`feature`, `bug`, `task`) and labels (`test`, `refine`, `review`, `docs`, `ambiguity`)
3. **Persona selection**: The system automatically selects the appropriate persona based on available work
4. **Iterative sessions**: Each session handles one issue, following the persona's protocol

## Quick Start

### 1. Initialize the workflow

```bash
./init-personas.sh
```

This creates:
- `instructions.md` - Main workflow instructions
- `run-personas.sh` - The automation loop script
- `.personas/` - Directory containing persona definitions

**Safe to re-run**: Skips files that already exist.

### 2. Create your specification

Write your project requirements in `specs.md`.

### 3. Run the workflow

```bash
./run-personas.sh
```

The script:
- Cycles through all available `opencode` models
- On timeout or failure, automatically tries the next model
- Wraps around to the first model when all have been tried
- Runs sessions until no more work is available

## Session Workflow

Each session follows these steps:

1. **Onboard**: `bd prime`
2. **Orient**: `git status`, `git log --oneline -5`, read `specs.md`
3. **Check project state**: `bd ready --json`, `bd list --status closed --json`
4. **Select persona**: Based on trigger conditions (see table above)
5. **Execute**: Follow the persona's protocol

## Personas

Trigger conditions are evaluated in order — the first match wins:

| Persona | Trigger | Responsibility |
|---------|---------|----------------|
| **Analyst** | `bd ready` is empty, OR ready issues tagged `ambiguity` | Extract requirements from specs, identify gaps, resolve ambiguities |
| **Tester** | Ready issues tagged `test` | Write and run tests, file bugs |
| **Refiner** | Ready issues tagged `refine` | Improve code quality, close gaps, handle edge cases |
| **Reviewer** | Ready issues tagged `review` | Evaluate codebase coherence, architecture, patterns |
| **Documentation** | Ready issues tagged `docs` | Write user-facing, developer, API, or operational docs |
| **Developer** | Ready issues: `feature`, `bug`, or untagged `task` | Implement features and fix bugs |

## Persona Details

### Developer
- **Unit tests**: Writes alongside implementation (happy path + error paths)
- **Shortcuts**: Logs deliberate shortcuts as `Refine:` issues with location and ideal approach
- **Out-of-scope**: Files discovered related work as new linked issues
- **Commit format**: `<type>(<scope>): <description>`

### Tester
- **Test types**: Unit (by Developer), Integration, E2E (Tester's focus)
- **Retest**: Must personally verify bug fixes before closing
- **Bug format**: Severity + steps to reproduce + expected vs actual + failing test name

### Refiner
- **Audit dimensions** (priority order): Correctness → Error handling → Edge cases → Clarity → Simplicity
- **Scope limit**: ~50 lines, few files — file remaining findings as linked issues
- **Architectural changes**: Files as `review` issue instead of implementing

### Reviewer
- **Checklist**: Correctness, Test coverage, Code style/clarity, Security, Documentation, Consistency, Recurring patterns
- **Recurring patterns**: Files single pattern-level issue (priority 1) instead of one per instance
- **No commit**: Only runs `git pull --rebase` and `git push`

### Documentation
- **Audiences**: User-facing, Developer-facing, API reference, Operational (do not mix)
- **Locations**: `docs/`, `docs/api/`, `docs/dev/`, `docs/ops/`
- **Discrepancies**: Files `Fix:` issues for missing/incorrect docstrings

### Analyst
- **Requirement categories**: Functional, Non-functional, Security, Error handling, Open questions
- **Gap types**: Missing issues, partial coverage, no test/refine linked, unverified acceptance, ambiguities
- **Auto-review**: Creates full review issue when all implementation closed but no review done
- **Ambiguity resolution**: Resolves from context (specs.md, codebase, notes) or requests human input
- **Terminal states**: `PROJECT COMPLETE` or `HUMAN INPUT NEEDED`

## Workflow Cycle

```
specs.md → Analyst creates issues → Developer implements → Tester verifies
                                           ↓
                          Refiner improves ← Reviewer evaluates
                                           ↓
                              Documentation writes docs
```

When all issues are closed, the Analyst re-evaluates specs.md for gaps. If none exist, the project is complete.

## Prerequisites

- **bash** (GNU coreutils for `timeout`)
- **opencode** CLI tool
- **bd** (backlog daemon) for issue tracking
- **git** for version control

## File Structure

```
project/
├── specs.md              # Project specifications (you create this)
├── instructions.md       # Generated workflow instructions
├── run-personas.sh       # Generated automation script
├── .personas/
│   ├── analyst.md
│   ├── developer.md
│   ├── tester.md
│   ├── refiner.md
│   ├── reviewer.md
│   └── documentation.md
└── .git/
```

## Key Commands

| Command | Description |
|---------|-------------|
| `bd ready --json` | List issues ready for work |
| `bd update <id> --claim` | Claim an issue for current session |
| `bd show <id> --json` | View issue details and notes |
| `bd list --status closed --json` | List all closed issues |
| `bd create "Title" -t task --labels refine` | Create a new issue |
| `bd note <id> "..."` | Add session notes to an issue |
| `bd update <id> --status closed` | Close an issue |

## Issue Types and Labels

**Types**: `feature`, `bug`, `task`

**Labels**: `test`, `refine`, `review`, `docs`, `ambiguity`

**Dependencies**: Issues link to related issues using `--deps discovered-from:<id>`

## Priority Reference

| Priority | Use Case |
|----------|----------|
| **1** (highest) | Ambiguity blocking work, recurring patterns |
| **2** | Test issues, review issues |
| **3** (lowest) | Refinement, documentation, discovered out-of-scope work |

## Session Rules

- **One issue per session**: Each persona handles exactly one issue, then stops
- **Session notes are handoff context**: Notes written by one session inform future sessions
- **Read only your persona**: Load and read only the selected persona file, not others
- **Do not skip steps**: Follow the persona protocol in order

## Session Note Formats

Each persona records session notes with specific information:

| Persona | Note Format |
|---------|-------------|
| Developer | "What was implemented, files changed, decisions made, shortcuts logged" |
| Tester | "Test types: \<unit/integration/E2E\>. Cases covered: \<list\>. All pass / Bugs filed: \<ids\>" |
| Refiner | "Improvement: \<what was changed and why\>. Remaining findings filed: \<ids\>" |
| Reviewer | "Scope: \<what was reviewed\>. Findings: \<count\> issues filed — \<ids\>. Patterns found: \<yes/no\>. Overall: \<assessment\>" |
| Documentation | "Documented: \<scope\>. Files created/updated: \<paths\>. Discrepancies filed: \<ids\>" |
| Analyst | "Resolution: \<what was decided and why, citing evidence\>" (for ambiguities) |

## Bug Severity

Tester classifies bugs as:

| Severity | Description |
|----------|-------------|
| **CRITICAL** | Data loss, security breach, or system-wide breakage |
| **MAJOR** | A feature is broken or an acceptance criterion fails |
| **MINOR** | Degraded behaviour, poor UX, non-blocking incorrect output |
| **TRIVIAL** | Cosmetic or negligible issues |

## Downstream Issue Patterns

After implementing an issue, the Developer automatically creates:

| Issue Type | Label | Priority | Purpose |
|------------|-------|----------|---------|
| `Refine: <title>` | `refine` | 3 | Code quality review |
| `Test: <title>` | `test` | 2 | Integration/E2E testing |
| `Document: <title>` | `docs` | 3 | User/developer documentation (if user-facing) |

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENCODE_TIMEOUT` | `1200` | Timeout in seconds for each AI session |

## run-personas.sh Behavior

**Preflight checks** (exits if any fail):
- `opencode` available in PATH
- `timeout` available (GNU coreutils)
- `instructions.md` exists in current directory
- At least one model returned by `opencode models`

**Runtime behavior**:
- Cycles through all available models
- On success: stays on same model, runs next session immediately
- On timeout (exit 124) or failure: advances to next model
- When all models exhausted: waits 30s, then cycles again
- Uses prompt: `read instructions.md and follow it`

**Troubleshooting**:
```bash
opencode auth list  # Check provider credentials
opencode models     # List available models
```

## Conventions

**Commits**: Conventional Commits format

| Persona | Commit Format |
|---------|---------------|
| Developer | `<type>(<scope>): <description>` (feat, fix, refactor) |
| Tester | `test: <description>` |
| Refiner | `refine: <description>` |
| Documentation | `docs: <description>` |
| Reviewer | No commit (sync only) |

**Git workflow**: Each session ends with `git pull --rebase` then `git push`

**Analyst terminal states**:
- `PROJECT COMPLETE` — All specs.md requirements have verified closed issues
- `HUMAN INPUT NEEDED` — Ambiguity requires human decision before proceeding

## License

MIT
