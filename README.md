# Personas

A workflow automation system for AI-assisted software development using specialized agent personas.

## Overview

Personas implements a structured development workflow where different AI agents take on specific roles (Developer, Tester, Refiner, Reviewer, Documentation, Analyst, Security Auditor, Investigator) to iteratively build software from specifications. Each persona has a focused responsibility and follows a defined protocol, ensuring systematic progress through the development lifecycle.

## How It Works

1. **Specifications-driven**: Work starts from a `specs.md` file that defines project requirements
2. **Change files**: Capabilities are tracked in `changes/<slug>.md` files, archived to `specs/` after review
3. **Issue tracking**: Uses `beads` (`bd`) to manage issues with types (`feature`, `bug`, `task`) and labels (`test`, `refine`, `review`, `docs`, `ambiguity`, `security`)
4. **Persona selection**: The system automatically selects the appropriate persona based on available work
5. **Iterative sessions**: Each session handles one issue, following the persona's protocol
6. **Memory system**: Personas use `bd remember` / `bd forget` to persist cross-session facts (build commands, conventions, hotspots, security patterns)

## Quick Start

### 1. Initialize the workflow

```bash
./init-personas.sh
```

This creates:
- `instructions.md` - Main workflow instructions
- `changes/` - Directory for change files (in-flight capabilities)
- `specs/` - Directory for archived specs (completed capabilities)
- `.personas/` - Directory containing persona definitions (developer, tester, refiner, reviewer, documentation, analyst, security, investigator)

**Safe to re-run**: Skips files that already exist.

### 2. Create your specification

Write your project requirements in `specs.md`.

### 3. Run the workflow

Start an AI agent session with the prompt:

```
read instructions.md and follow it
```

Each session reads `instructions.md`, orients via `bd ready --json`, selects the appropriate persona, and executes its protocol. Run sessions iteratively until no more work is available.

## Session Workflow

Each session follows these steps:

1. **Onboard**: `bd prime`
2. **Orient**: `git status`, `git log --oneline -5`
   - If uncommitted changes exist: commit or stash before proceeding
3. **Check project state**: `bd ready --json`
4. **Select persona**: Scan tags of ready issues; read only the `# TRIGGER` section of each persona file (first block only) to confirm selection
5. **Load context**: `bd show <id> --json` fully (including all notes from previous sessions), read the change file referenced in the issue description (for `feature`, `bug`, or untagged `task` issues, a missing change file reference is treated as malformed — the agent stops and waits for human input); for all other issue types, `specs.md` is used as fallback context
6. **Execute**: Follow the persona's protocol
7. **Record**: Write a session note with a status token (e.g. `[impl] DONE — ...`), then commit and stop

## Personas

Trigger conditions are evaluated in order — the first match wins:

| # | Persona | Trigger | Responsibility |
|---|---------|---------|----------------|
| 1 | **Analyst** | `bd ready` is empty | Extract requirements from specs, identify gaps, resolve ambiguities |
| 2 | **Analyst** | Ready issues tagged `ambiguity` | Resolve ambiguities from context or request human input |
| 3 | **Security Auditor** | Ready issues tagged `security` | Audit code for vulnerabilities, file findings |
| 4 | **Reviewer** | Ready issues tagged `review` | Evaluate codebase coherence, architecture, patterns; archive change files |
| 5 | **Tester** | Ready issues tagged `test` | Write and run integration/E2E tests, file bugs |
| 6 | **Investigator** | Ready `bug` issues without `root-cause:` note | Diagnose root cause, create derived bug with diagnosis |
| 7 | **Developer** | Ready issues: `feature`, `bug` (with `root-cause:`), or untagged `task` | Implement features and fix bugs |
| 8 | **Refiner** | Ready issues tagged `refine` | Improve code quality, close gaps, handle edge cases |
| 9 | **Documentation** | Ready issues tagged `docs` | Write user-facing, developer, API, or operational docs |

## Persona Details

### Developer
- **Role**: Implement only — does not evaluate quality, does not refactor existing work
- **Change file**: Creates `changes/<slug>.md` before implementing (Why, Scope, Out of scope, Constraints, Decision log, Open questions); As built filled in by Reviewer
- **Code quality**: SOLID principles, composition over inheritance, meaningful names, small functions, single responsibility, minimal coupling; object calisthenics rules
- **Avoid**: Hidden global state
- **Docstrings**: Every public function must have a docstring describing its contract
- **Unit tests**: Writes alongside implementation (happy path + error paths) — does not defer to Tester
- **Completeness**: Defaults to complete implementation; defers only when genuinely blocked (missing information, out-of-scope decision, unbuilt dependency)
- **Shortcuts**: Logs deliberate shortcuts as `Refine:` issues with location, ideal approach, and **reason complete version is blocked**
- **Out-of-scope**: Files discovered related work as new linked issues; never silently skip requirements; implement exactly what the issue describes, no more
- **Bug handling**: Checks for `root-cause:` note from Investigator; if missing, derives root cause before coding
- **Test failure protocol**: After 3 failed fix attempts, stops and files investigation issue
- **Commit format**: `<type>(<scope>): <description>` (feat, fix, refactor)
- **Downstream issues**: Creates `Refine:`, `Test:`, and optionally `Document:` issues before closing
- **Memory**: Stores actionable project facts via `bd remember` before committing

### Tester
- **Test types**: Unit (by Developer), Integration, E2E (Tester's focus)
- **Test plan**: Derived from change file's Scope/Constraints and parent issue notes
- **Approach**: Skeptical — goal is to find failures, not confirm success
- **Test cases**: Happy path, boundary conditions, invalid inputs, error paths, edge cases, things the developer assumed would never happen
- **Tests must be**: Deterministic
- **Passing criteria**: Never marks a component passed unless ALL acceptance criteria are verified; a passing unit test suite is a floor, not a ceiling
- **Test infrastructure escalation cap**: After 3 failed attempts to stabilize test infrastructure, stops and files investigation issue
- **Retest**: Must personally verify bug fixes before closing; for `bug`-type parents, confirms the specific failure no longer occurs (not just that tests pass)
- **Security re-audit**: After tests pass on a `security`-labeled bug fix, creates a security re-audit task
- **Bug severity/priority mapping**: CRITICAL→p1, MAJOR→p2, MINOR→p3, TRIVIAL→p4
- **Bug format**: `Change file: changes/<slug>.md` + Severity + steps to reproduce + expected vs actual + failing test name
- **Does not fix bugs**: Creates issues for Developer to fix
- **Does not write implementation code**: Under no circumstance
- **Memory**: Stores testing facts, infrastructure quirks, edge-case patterns via `bd remember`
- **Session note**: `[test] STATUS: <token> — Test types: <...>. Cases covered: <...>. Result: <...>. Bugs filed: <...>`

### Refiner
- **Audit dimensions** (priority order): Correctness → Error handling → Edge cases → Clarity → Simplicity
- **Scope limit**: ~50 lines, few files — if you exceed this, you have scope-crept; file remaining findings as linked issues
- **Architectural changes**: Files as `review` issue instead of implementing
- **Evidence-based**: Every proposal cites file:line, change file requirement, or Developer notes; vague proposals not acceptable
- **Restraint**: One focused improvement per session; never propose changes conflicting with evident design decisions without creating review issue first; consults `## Decision log` before concluding something is a defect
- **Process**: Lists every finding before acting on any of them
- **Fix failure protocol**: After 2 failed fix attempts, reverts change entirely and files investigation issue; selects next finding
- **Decision log awareness**: Reads change file's `## Decision log` before flagging something as a defect
- **Hard limit**: Does not close the issue if any test fails after a change
- **Memory**: Stores debt patterns, hotspot files, convention drift observations via `bd remember`
- **Session note**: `[refine] STATUS: <token> — Improvement: <...>. Remaining findings filed: <...>`

### Reviewer
- **Checklist**: Correctness, Test coverage, Code style/clarity, Security, Documentation, Consistency, Recurring patterns
- **Recurring patterns**: Files single pattern-level issue (priority 1) instead of one per instance
- **Specialist lenses**: Performance (N+1 queries, missing indexes, unbounded results, frontend waterfalls/re-renders, missing pagination), Maintainability (dead code, magic numbers, stale comments, duplicated literals), API contract (parameter docs, versioning, error format consistency, required field migration)
- **Archives change files**: Moves `changes/<slug>.md` to `specs/<slug>.md` after review only when no findings were filed (fills in As built section); if findings were filed, change file stays in `changes/`
- **Review scope**: Reads change file, closed issue notes, source files, and settled specs in `specs/` for adjacent capabilities; treats `specs/` as decision history, not frozen specs
- **Open questions check**: Verifies all open questions from change file were filed or resolved; files unresolved ones as ambiguity issues
- **Does not modify**: Source code under any circumstance
- **Does not archive**: If blocking findings remain open — change file stays in `changes/`
- **Does not treat `specs/` as obsolete**: When `specs.md` drifts, `specs/` files are decision history, not errors to fix
- **Capability readiness**: Outputs readiness summary (Security audit, Tests, Refine passes, Review, Archived) before closing
- **Commit format**: `review(<scope>): <one line summary>`
- **Memory**: Stores structural observations, architecture coupling, consistency gaps via `bd remember`
- **Session note**: `[review] STATUS: <token> — <capability readiness summary>`

### Documentation
- **Audiences**: User-facing, Developer-facing, API reference, Operational (do not mix)
- **Default audience**: User-facing (if ambiguous)
- **Locations**: `docs/`, `docs/api/`, `docs/dev/`, `docs/ops/`
- **Existing doc check**: Checks for existing docs before writing; updates stale docs rather than creating parallels
- **Discrepancies**: Files `Fix:` issues for missing/incorrect docstrings; escalates spec-vs-code correctness gaps as bugs
- **Does not**: Implement features or refactor code; only reads what exists and writes clearly
- **Accuracy**: Never documents behaviour not verified by reading actual implementation
- **Style**: Plain language; lead with what something does before explaining how; use examples for non-obvious behaviour
- **Focus**: Explains intent, not mechanics; answers: what does this do, when should I use it, what can go wrong, what does the output look like
- **Verification**: Re-reads relevant code after writing; confirm every claim (parameter names, return types, error conditions, default values, conditional behaviour)
- **Memory**: Stores doc debt, volatility notes, audience context via `bd remember`
- **Session note**: `[docs] STATUS: <token> — Documented: <...>. Files created/updated: <...>. Discrepancies filed: <...>`

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
- **Retrospective health check** (empty queue mode): Analyzes last 14 days of commits — test ratio below 20% triggers memory, files with 3+ `refine(` commits become hotspot review issues, capabilities with 14+ days of no activity get stall notes
- **Review readiness summary**: Outputs capability-level table showing Security/Test/Refine/Review status before listing gaps
- **Convergence check**: Before declaring project complete, verifies no closed issues have final notes containing `STATUS: BLOCKED` or `STATUS: NEEDS_CONTEXT` (silently stuck issues must be resolved first)
- **specs.md as moving target**: Treats `specs/` as decision history, not frozen specs; notes drift as context rather than fixing divergence

### Security Auditor
- **Role**: Find vulnerabilities — not to fix them; reads code with adversarial eyes
- **Audit categories**: Input validation, Injection (SQL, command, path traversal, template, SSRF, LDAP, header), Authentication & authorization (incl. direct object reference, session expiry), Cryptographic misuse (weak hashing, non-CSPRNG, timing attacks, hardcoded keys), Secrets & credentials, XSS escape hatches (Rails `.html_safe`, React `dangerouslySetInnerHTML`, Vue `v-html`, Django `|safe`, `innerHTML`), Deserialization (pickle, Marshal, unsafe YAML), Error handling & info leakage, Dependencies, Data handling
- **Scope**: Reads change file scope/constraints/decision log, relevant source files, and adjacent settled specs for trust boundary analysis
- **Severity/priority mapping**: CRITICAL→p1, HIGH→p1, MEDIUM→p2, LOW→p2
- **Finding format**: Severity + Location (file:line) + attack vector + concrete impact + suggested fix; `Change file: changes/<slug>.md` mandatory
- **Architectural findings**: Files as `review` issue instead of bug when trust boundary redesign needed
- **Does not modify**: Source code, no refactoring
- **No vague findings**: Every issue must include location, attack vector, and concrete impact
- **No silent passes**: Does not mark a component secure without auditing every category in the checklist
- **No downgrades**: Security findings are always priority 1 or 2 — never filed as `refine` issues
- **Memory**: Stores security patterns, trust boundaries, recurring insecure patterns via `bd remember`
- **Session note**: `[security] STATUS: <token> — Categories audited: <...>. Findings: <count> — <ids>. Clean categories: <...>`

### Investigator
- **Role**: Root cause diagnosis for bugs lacking a `root-cause:` note — not implementation
- **Process**: Gathers symptoms (error output, reproduction steps), traces code path backwards from symptom, forms and verifies hypotheses
- **Hypothesis verification**: Must point to specific file:line where failure originates; must explain why that location produces observed symptom
- **3-strike rule**: After 3 failed hypotheses, stops and files ambiguity issue requiring runtime information
- **Output**: Closes original bug with diagnosis note (`STATUS: DONE` on success, `STATUS: BLOCKED` on 3-strike), creates derived bug with `root-cause:` field populated in description (what dispatch table checks for Developer)
- **Fix recommendation**: Optional, advisory only — includes confidence level and caveats; Developer must verify independently
- **Does not**: Write or modify production code, close original bug without creating derived diagnosis
- **Memory**: Stores diagnostic patterns, fragile areas, debug tips via `bd remember`
- **Session note**: `[investigate] STATUS: DONE — Root cause: <...>. Fix recommendation: <if applicable>. Evidence: <...>.`

## Workflow Cycle

```
specs.md → Analyst creates change file + issues → Developer implements → Tester verifies
                                                        ↓
                               Refiner improves ← Reviewer evaluates & archives
                                                        ↓
                                           Documentation writes docs

Parallel tracks:
  Investigator → diagnoses bugs without root-cause → creates derived bug for Developer
  Security Auditor → audits code for vulnerabilities → files bugs for Developer
```

When all issues are closed, the Analyst re-evaluates specs.md for gaps. If none exist, the project is complete.

## Prerequisites

- **bash**
- An AI agent capable of reading files and running shell commands (e.g. opencode, Claude Code, etc.)
- **beads** (`bd`) for issue tracking
- **git** for version control

## File Structure

```
project/
├── specs.md              # Project specifications (you create this)
├── instructions.md       # Generated workflow instructions
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
│   ├── documentation.md
│   ├── security.md
│   └── investigator.md
└── .git/
```

## Key Commands

| Command | Description |
|---------|-------------|
| `bd prime` | Initialize the issue tracking database |
| `bd ready --json` | List issues ready for work |
| `bd update <id> --claim` | Claim an issue for current session |
| `bd show <id> --json` | View issue details and notes |
| `bd list --status closed --json` | List all closed issues |
| `bd list --status open --labels ambiguity --json` | Check for unresolved ambiguities |
| `bd create "Title" -t task --labels refine` | Create a new issue |
| `bd note <id> "..."` | Add session notes to an issue |
| `bd update <id> --status closed` | Close an issue |
| `bd remember "fact"` | Persist a cross-session fact |
| `bd forget <key>` | Remove a stale memory |

## Issue Types and Labels

**Types**: `feature`, `bug`, `task`

**Labels**: `test`, `refine`, `review`, `docs`, `ambiguity`, `security`

**Dependencies**: Issues spawned from an existing issue link back using `--deps discovered-from:<id>` (Analyst-created issues during gap analysis are top-level and do not use dependencies)

## Priority Reference

| Priority | Use Case |
|----------|----------|
| **1** (highest) | Ambiguity blocking work, recurring patterns, security findings (CRITICAL/HIGH), test infrastructure failures, stalled test fixes |
| **2** | Test issues, review issues, security findings (MEDIUM/LOW), security re-audit tasks |
| **3** (low) | Refinement, documentation, discovered out-of-scope work, docstring fixes |
| **4** (lowest) | Trivial bug findings from Tester |

## Session Rules

- **One issue per session**: Each persona handles exactly one issue, then stops. (Exception: Analyst in empty-queue mode may create multiple issues during gap analysis.)
- **Session notes are handoff context**: Notes written by one session inform future sessions
- **Read only your persona**: Load and read only the selected persona file, not others
- **Do not skip steps**: Follow the persona protocol in order

## Session Note Formats

Each persona records session notes with a type prefix and status token:

| Persona | Note Format |
|---------|-------------|
| Developer | `[impl] STATUS: <token> — What was implemented, files changed, decisions made, shortcuts logged` |
| Tester | `[test] STATUS: <token> — Test types: \<unit/integration/E2E\>. Cases covered: \<list\>. Result: all pass / Bugs filed: \<ids\>` |
| Refiner | `[refine] STATUS: <token> — Improvement: \<what was changed and why\>. Remaining findings filed: \<ids\>` |
| Reviewer | `[review] STATUS: <token> — <capability readiness summary>` |
| Documentation | `[docs] STATUS: <token> — Documented: \<scope\>. Files created/updated: \<paths\>. Discrepancies filed: \<ids if any\>` |
| Analyst (ambiguity) | `[analyst] STATUS: DONE — Resolution: \<what was decided and why, citing evidence\>` or `[analyst] STATUS: NEEDS_CONTEXT — Unresolvable autonomously. Human input required.` |
| Security Auditor | `[security] STATUS: <token> — Categories audited: \<list\>. Findings: \<count\> — \<ids\>. Clean categories: \<list\>` |
| Investigator | `[investigate] STATUS: DONE — Root cause: \<one sentence\>. Fix recommendation: \<if applicable\>. Evidence: \<file:line and explanation\>.` |

**Status tokens**: `DONE`, `DONE_WITH_CONCERNS`, `BLOCKED`, `NEEDS_CONTEXT`

**Note**: Analyst triggered by empty queue does not write session notes — it either creates issues or outputs terminal state.

## Bug Severity

### Tester severity (general bugs)

| Severity | Description |
|----------|-------------|
| **CRITICAL** | Data loss, security breach, or system-wide breakage |
| **MAJOR** | A feature is broken or an acceptance criterion fails |
| **MINOR** | Degraded behaviour, poor UX, non-blocking incorrect output |
| **TRIVIAL** | Cosmetic or negligible issues |

### Security Auditor severity (vulnerability findings)

| Severity | Description | Priority |
|----------|-------------|----------|
| **CRITICAL** | Direct exploit path: RCE, auth bypass, data exfiltration | 1 |
| **HIGH** | Significant risk requiring non-trivial exploitation | 1 |
| **MEDIUM** | Limited impact or requires specific conditions to exploit | 2 |
| **LOW** | Hardening improvement, defense-in-depth, minimal direct risk | 2 |

## Downstream Issue Patterns

After implementing an issue, the Developer automatically creates:

| Issue Type | Label | Priority | Purpose |
|------------|-------|----------|---------|
| `Refine: <title>` | `refine` | 3 | Code quality review (gaps, error handling, edge cases) |
| `Test: <title>` | `test` | 2 | Integration/E2E testing (unit tests written by Developer) |
| `Document: <title>` | `docs` | 3 | Documentation for user-facing behaviour, public APIs, or operational considerations (skipped for purely internal features) |

All downstream issues reference the change file: `Change file: changes/<slug>.md`

After refining, the Refiner creates:

| Issue Type | Label | Priority | Purpose |
|------------|-------|----------|---------|
| `Test: <title>` | `test` | 2 | Verify behavioural improvement made by Refiner |
| `Refine: <title>` | `refine` | 3 | Remaining findings not addressed in this session |

After security audit, the Security Auditor creates:

| Issue Type | Label | Priority | Purpose |
|------------|-------|----------|---------|
| `Bug: <title>` | `security` | 1-2 | Vulnerability findings with attack vector and impact |
| `Review: <title>` | `review` | 1 | Architectural security decisions requiring review |

After testing, the Tester creates (for `security`-labeled parent fixes):

| Issue Type | Label | Priority | Purpose |
|------------|-------|----------|---------|
| `Security: re-audit fix for <id>` | `security` | 1 | Verify security fix is sound after tests pass |

During review, the Reviewer may create:

| Issue Type | Label | Priority | Purpose |
|------------|-------|----------|---------|
| `Bug: <title>` / `Task: <title>` | `refine`/`test`/`docs`/`security` | varies | Standard findings from review checklist |
| `Refine: recurring pattern — <n>` | `refine` | 1 | Same defect type in multiple locations |
| `Ambiguity: <topic>` | `ambiguity` | 1 | Requirements gaps discovered during review |

## Conventions

**Commits**: Conventional Commits format with scope as change file slug

| Persona | Commit Format |
|---------|---------------|
| Developer | `<type>(<scope>): <description>` (feat, fix, refactor) |
| Tester | `test(<scope>): <description>` |
| Refiner | `refine(<scope>): <description>` |
| Reviewer | `review(<scope>): <one line summary>` |
| Documentation | `docs(<scope>): <description>` |
| Security Auditor | `security(<scope>): <one line summary>` |
| Investigator | `investigate(<scope>): <short description of root cause found>` |
| Analyst | `chore(analyst): <short description of gaps identified or work created>` |

**Types used across personas**: `feat`, `fix`, `refine`, `test`, `review`, `docs`, `security`, `investigate`, `chore`

**Memory system**: All personas use `bd remember` to persist cross-session facts and `bd forget` to correct stale memories. Facts stored include build commands, conventions, fragile test infrastructure, refine hotspots, security patterns, diagnostic tips, doc debt, and architectural observations.

**Analyst terminal states**:
- `PROJECT COMPLETE` — Outputs: "All requirements in specs.md are covered by settled specs in specs/. All change files have been archived. No ambiguities remain unresolved. bd ready is empty. No further sessions needed."
- `HUMAN INPUT NEEDED` — Outputs: "Ambiguity: <topic>", "Question: <the exact decision that must be made>", "Context: <what specs.md says, what specs/ history shows, what has been assumed, what is at stake>", "Conflict: <if applicable — two contradicting statements and their sources>"

## License

MIT
