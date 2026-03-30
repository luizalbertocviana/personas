# Personas

A workflow automation system for AI-assisted software development using specialized agent personas.

## Overview

Personas implements a structured development workflow where different AI agents take on specific roles (Developer, Tester, Refiner, Reviewer, Documentation Specialist, Analyst) to iteratively build software from specifications. Each persona has a focused responsibility and follows a defined protocol, ensuring systematic progress through the development lifecycle.

## How It Works

1. **Specifications-driven**: Work starts from a `specs.md` file that defines project requirements
2. **Change files**: Capabilities are tracked in `changes/<slug>.md` files, archived to `specs/` after review
3. **Issue tracking**: Uses `beads` (`bd`) to manage issues with types (`feature`, `bug`, `task`) and labels (`test`, `refine`, `review`, `docs`, `ambiguity`)
4. **Persona selection**: The system automatically selects the appropriate persona based on available work
5. **Iterative sessions**: Each session handles one issue, following the persona's protocol

## Quick Start

### 1. Initialize the workflow

```bash
./init-personas.sh
```

This creates:
- `instructions.md` - Main workflow instructions
- `run-personas.sh` - The automation loop script
- `changes/` - Directory for change files (in-flight capabilities)
- `specs/` - Directory for archived specs (completed capabilities)
- `.personas/` - Directory containing persona definitions

**Safe to re-run**: Skips files that already exist.

### 2. Create your specification

Write your project requirements in `specs.md`.

### 3. Run the workflow

```bash
./run-personas.sh
```

The script:
- Cycles through all available `opencode` models (or use `OPENCODE_MODELS="model1 model2" ./run-personas.sh` to specify)
- On timeout or failure, automatically tries the next model
- Wraps around to the first model when all have been tried
- Runs sessions until no more work is available

## Session Workflow

Each session follows these steps:

1. **Onboard**: `bd prime`
2. **Orient**: `git status`, `git log --oneline -5`
   - If uncommitted changes exist: commit or stash before proceeding
3. **Check project state**: `bd ready --json`
4. **Select persona**: Scan tags of ready issues; read only the `# TRIGGER` section of each persona file (first block only) to confirm selection
5. **Load context**: `bd show <id> --json` fully (including all notes from previous sessions), read change file or `specs.md`
6. **Execute**: Follow the persona's protocol

## Personas

Trigger conditions are evaluated in order — the first match wins:

| # | Persona | Trigger | Responsibility |
|---|---------|---------|----------------|
| 1 | **Analyst** | `bd ready` is empty | Extract requirements from specs, identify gaps, resolve ambiguities |
| 2 | **Analyst** | Ready issues tagged `ambiguity` | Resolve ambiguities from context or request human input |
| 3 | **Tester** | Ready issues tagged `test` | Write and run tests, file bugs |
| 4 | **Refiner** | Ready issues tagged `refine` | Improve code quality, close gaps, handle edge cases |
| 5 | **Reviewer** | Ready issues tagged `review` | Evaluate codebase coherence, architecture, patterns |
| 6 | **Documentation** | Ready issues tagged `docs` | Write user-facing, developer, API, or operational docs |
| 7 | **Developer** | Ready issues: `feature`, `bug`, or untagged `task` | Implement features and fix bugs |

## Persona Details

### Developer
- **Role**: Implement only — does not evaluate quality, does not refactor existing work
- **Change file**: Creates `changes/<slug>.md` before implementing (Why, Scope, Out of scope, Constraints, Open questions); As built filled in by Reviewer
- **Code quality**: SOLID principles, composition over inheritance, meaningful names, small functions, single responsibility, minimal coupling
- **Avoid**: Hidden global state
- **Docstrings**: Every public function must have a docstring describing its contract
- **Unit tests**: Writes alongside implementation (happy path + error paths) — do not defer to Tester
- **Shortcuts**: Logs deliberate shortcuts as `Refine:` issues with location and ideal approach
- **Out-of-scope**: Files discovered related work as new linked issues; never silently skip requirements; implement exactly what the issue describes, no more
- **Commit format**: `<type>(<scope>): <description>` (feat, fix, refactor)
- **Downstream issues**: Creates `Refine:`, `Test:`, and optionally `Document:` issues before closing

### Tester
- **Test types**: Unit (by Developer), Integration, E2E (Tester's focus)
- **Test plan**: Derived from change file's Scope/Constraints and parent issue notes
- **Approach**: Skeptical — goal is to find failures, not confirm success
- **Test cases**: Happy path, boundary conditions, invalid inputs, error paths, edge cases, things the developer assumed would never happen
- **Tests must be**: Deterministic
- **Passing criteria**: Never mark a component passed unless ALL acceptance criteria are verified; a passing unit test suite is a floor, not a ceiling
- **Retest**: Must personally verify bug fixes before closing
- **Bug format**: Severity + steps to reproduce + expected vs actual + failing test name
- **Does not fix bugs**: Creates issues for Developer to fix

### Refiner
- **Audit dimensions** (priority order): Correctness → Error handling → Edge cases → Clarity → Simplicity
- **Scope limit**: ~50 lines, few files — file remaining findings as linked issues
- **Architectural changes**: Files as `review` issue instead of implementing
- **Evidence-based**: Every proposal cites file:line, change file requirement, or Developer notes; vague proposals not acceptable
- **Restraint**: One focused improvement per session; never propose changes conflicting with evident design decisions without creating review issue first
- **Process**: List every finding before acting on any of them

### Reviewer
- **Checklist**: Correctness, Test coverage, Code style/clarity, Security, Documentation, Consistency, Recurring patterns
- **Recurring patterns**: Files single pattern-level issue (priority 1) instead of one per instance
- **Archives change files**: Moves `changes/<slug>.md` to `specs/<slug>.md` after review (fills in As built section)
- **Review scope**: Reads change file, closed issue notes, source files, and settled specs in `specs/` for adjacent capabilities
- **Does not fix**: Only produces findings and archives change files
- **Commit format**: `review: <slug> — <one line summary>`

### Documentation
- **Audiences**: User-facing, Developer-facing, API reference, Operational (do not mix)
- **Default audience**: User-facing (if ambiguous)
- **Locations**: `docs/`, `docs/api/`, `docs/dev/`, `docs/ops/`
- **Discrepancies**: Files `Fix:` issues for missing/incorrect docstrings
- **Does not**: Implement features or refactor code; only reads what exists and writes clearly
- **Accuracy**: Never documents behaviour not verified by reading actual implementation
- **Style**: Plain language; lead with what something does before explaining how; use examples for non-obvious behaviour
- **Focus**: Explains intent, not mechanics; answers: what does this do, when should I use it, what can go wrong, what does the output look like
- **Verification**: Re-read relevant code after writing; confirm every claim (parameter names, return types, error conditions, default values, conditional behaviour)

### Analyst
- **Trigger modes**: Empty queue (gap analysis) OR ambiguity issues (resolution)
- **Requirement categories**: Functional, Non-functional, Security, Error handling, Open questions
- **Gap types**: Missing issues, partial coverage, no test/refine linked, unverified acceptance, ambiguities
- **Auto-review**: Creates full review issue when all implementation closed but no review done
- **Ambiguity resolution**: Resolves from context (specs.md, specs/, codebase, closed issue notes); does not invent requirements
- **Terminal states**: `PROJECT COMPLETE` or `HUMAN INPUT NEEDED`
- **Change files**: Writes `changes/<slug>.md` for new capabilities before creating issues
- **Gap analysis**: Reads specs.md, all files in specs/, all files in changes/, full issue history, and inspects codebase
- **Ambiguity handling**: After resolving or surfacing ambiguity, stops immediately; does not attempt any other work

## Workflow Cycle

```
specs.md → Analyst creates change file + issues → Developer implements → Tester verifies
                                                        ↓
                               Refiner improves ← Reviewer evaluates & archives
                                                        ↓
                                           Documentation writes docs
```

When all issues are closed, the Analyst re-evaluates specs.md for gaps. If none exist, the project is complete.

## Prerequisites

- **bash** (GNU coreutils for `timeout`)
- **opencode** CLI tool
- **beads** (`bd`) for issue tracking
- **git** for version control

## File Structure

```
project/
├── specs.md              # Project specifications (you create this)
├── instructions.md       # Generated workflow instructions
├── run-personas.sh       # Generated automation script
├── changes/              # Change files for in-flight capabilities
│   └── .gitkeep
├── specs/                # Archived specs for completed capabilities
│   └── .gitkeep
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
| Tester | "Test types: \<unit/integration/E2E\>. Cases covered: \<list\>. Result: all pass / Bugs filed: \<ids\>" |
| Refiner | "Improvement: \<what was changed and why\>. Remaining findings filed: \<ids\>" |
| Reviewer | "Scope reviewed: \<change file\>. Findings: \<count\> issues — \<ids\>. Archived: \<yes/no, reason if no\>" |
| Documentation | "Documented: \<scope\>. Files created/updated: \<paths\>. Discrepancies filed: \<ids if any\>" |
| Analyst (ambiguity) | "Resolution: \<what was decided and why, citing evidence\>" or "Unresolvable autonomously. Human input required." |

**Note**: Analyst triggered by empty queue does not write session notes — it either creates issues or outputs terminal state.

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
| `Refine: <title>` | `refine` | 3 | Code quality review (gaps, error handling, edge cases) |
| `Test: <title>` | `test` | 2 | Integration/E2E testing (unit tests written by Developer) |
| `Document: <title>` | `docs` | 3 | User/developer documentation (if user-facing) |

All downstream issues reference the change file: `Change file: changes/<slug>.md`

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENCODE_TIMEOUT` | `300` | Timeout in seconds for each AI session |
| `OPENCODE_MODELS` | (all models) | Space-separated list of models to use (e.g., `"anthropic/claude-sonnet-4-5 openai/gpt-4o"`) |

## run-personas.sh Behavior

**Preflight checks** (exits if any fail):
- `opencode` available in PATH
- `timeout` available (GNU coreutils)
- `instructions.md` exists in current directory
- Models available: from `opencode models` or `OPENCODE_MODELS` environment variable

**Runtime behavior**:
- Uses all models from `opencode models`, or the subset specified in `OPENCODE_MODELS`
- On success: stays on same model, runs next session immediately
- On timeout (exit 124): advances to next model with message "timed out after ${TIMEOUT_SECONDS}s (likely hung on quota/rate limit)"
- On failure (other exit): advances to next model with exit code
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
| Reviewer | `review: <slug> — <one line summary>` |
| Documentation | `docs: <description>` |

**Git workflow**: Each session ends with `git pull --rebase` then `git push`

**Analyst terminal states**:
- `PROJECT COMPLETE` — Outputs: "All requirements in specs.md are covered by settled specs in specs/. All change files have been archived. No ambiguities remain unresolved. bd ready is empty. No further sessions needed."
- `HUMAN INPUT NEEDED` — Outputs: "Ambiguity: <topic>", "Question: <the exact decision that must be made>", "Context: <what specs.md says, what has been assumed, what is at stake>"

## License

MIT
