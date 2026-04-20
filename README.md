<h1 align="center">better-skills</h1>
<p align="center"><a href="docs/README_ko.md">한국어 문서 (Korean)</a></p>

An autonomous development pipeline plugin for Claude Code.

Feed it a spec, and it automatically handles planning, TDD development, verification, and skill maintenance.

```
/dev Implement a user authentication system. I need login, signup, and password reset.
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI must be installed.

## Installation

```bash
# Install with project scope (available only in the current project)
claude plugin install /path/to/better-skills --scope project

# Install with user scope (available across all projects)
claude plugin install /path/to/better-skills

# Install directly from GitHub
claude plugin install github:ohdowon064/better-skills --scope project
```

After installation, Claude Code automatically recognizes the `skills/` and `agents/` directories.

## Quick Start

### Develop an entire feature end-to-end

```bash
# Pass a spec as text
/dev Implement a user authentication system. I need login, signup, and password reset.

# Pass a markdown file
/dev docs/auth-spec.md

# Resume interrupted work
/dev PLAN_auth
```

You only need to intervene at 3 points:

1. **Plan approval** — Confirm the phase breakdown is appropriate
2. **FAIL action selection** — Auto-fix / individual review / skip
3. **Unexpected blockers** — Decide on resolution

### Call individual skills directly

```bash
# Create a plan only
/feature-planner

# Update an existing plan
/feature-planner update PLAN_auth

# Mark a plan as complete
/feature-planner complete PLAN_auth

# Run verification only
/verify-implementation
/verify-implementation phase-1
/verify-implementation --current-phase

# Check skills
/manage-skills

# Code review
/code-review
/code-review --staged
/code-review --branch feature/auth
```

## Pipeline Flow

```
Spec Input → Planning → TDD per Phase → Verification → Code Review → Integration Review → Completion → Skill Cleanup
                │              │            │              │              │
                │              │            │              │              └─ code-reviewer (integration)
                │              │            │              └─ code-reviewer (phase)
                │              │            └─ verify-implementation
                │              └─ RED → GREEN → REFACTOR
                └─ feature-planner
```

`/dev` automatically orchestrates this entire flow.

## Skills

| Skill | Description |
|-------|-------------|
| `/dev` | Full pipeline orchestrator. Spec → finished feature, fully automated |
| `/feature-planner` | Breaks features into 3-7 phases. CREATE / UPDATE / COMPLETE modes |
| `/verify-implementation` | Runs verify skills in parallel via subagents, generates unified verification report |
| `/manage-skills` | Detects and fixes verify skill drift in response to code changes |
| `/code-review` | Code review. Supports recent changes, specific paths, staged, and branch diffs |

## Subagents

| Agent | Role | Used By |
|-------|------|---------|
| `codebase-scanner` | Unified project environment analysis (structure, tests, linters, CI/CD, git changes) | feature-planner, manage-skills |
| `skill-writer` | Creates/updates verify skills (parallel execution) | feature-planner, manage-skills |
| `test-runner` | Runs individual verify skills + TDD order validation | verify-implementation |
| `code-reviewer` | Design/security/performance review (per-phase + integration) | dev, code-review |

## Key Concepts

### Built-in TDD

TDD is not a recommendation — it's a mandatory pipeline stage.

- Every phase follows the RED → GREEN → REFACTOR cycle
- Test-first order is automatically verified via git history
- REFACTOR effectiveness is quantitatively measured (file length, function count, average function length, duplication)

### Skill Lifecycle

Skill status is determined by registry presence:

- **In registry** = active (target for verify-implementation)
- **Not in registry** = inactive (history preserved in git)

When a phase completes, `verify-phase-N-*` skills are **consolidated** into general `verify-*` skills (new skill created + old ones removed). Unnecessary skills are deleted from both the registry and filesystem; they can be recovered via `git log` if needed.

### Automatic Exception Suggestions

When you skip an issue during verification, you're immediately asked whether to add that check to the skill's Exceptions. If approved, the skill is auto-updated so the same false positive doesn't recur.

### Automatic Project Environment Detection

`codebase-scanner` automatically identifies the repo type:

| Type | Condition |
|------|-----------|
| BRAND_NEW | Git not initialized or 0 commits |
| FRESH_START | 5 or fewer commits, no source code |
| CONFIG_ONLY | Has commits, only config files exist |
| EXISTING | Source code exists |

Detected actual commands populate the Quality Gate — not placeholders like `npm test`, but the project's real commands.

### SSOT Registry

`.claude/skill-registry.json` is the single source of truth for verify skills. Both verify-implementation and feature-planner reference this file at runtime.

## Directory Structure

### Plugin (this repo)

```
better-skills/
├── .claude-plugin/
│   ├── plugin.json                     # Plugin manifest
│   └── marketplace.json                # Marketplace listing
├── skills/
│   ├── dev/SKILL.md                    # Pipeline orchestrator
│   ├── feature-planner/
│   │   ├── SKILL.md                    # Planning
│   │   └── plan-template.md            # Plan document template
│   ├── verify-implementation/SKILL.md  # Unified verification engine
│   ├── manage-skills/SKILL.md          # Skill maintenance
│   └── code-review/SKILL.md            # Code review (user-invoked)
├── agents/
│   ├── codebase-scanner.md             # Project context analysis
│   ├── skill-writer.md                 # Verify skill creation/update
│   ├── test-runner.md                  # Verification execution (parallel)
│   └── code-reviewer.md               # Code review (design/security/performance)
```

### Files generated at runtime in the target project

```
your-project/
├── .claude/
│   ├── skills/verify-*/SKILL.md        # Auto-generated verify skills
│   └── skill-registry.json             # Skill registry SSOT
└── docs/plans/PLAN_*.md                # Plan documents
```

## Usage Examples

### Starting from an empty project

```bash
mkdir my-app && cd my-app && git init
claude plugin install /path/to/better-skills --scope project

/dev Build a backend API for a Todo app.
# → Tech stack selection → Phase breakdown → Approval → Auto development
```

### Adding a feature to an existing project

```bash
cd my-existing-app
claude plugin install /path/to/better-skills --scope project

/dev Add search functionality. I need keyword search across titles and body text.
# → Existing code analysis → Follow existing patterns → Phase breakdown → Auto development
```

### Parallel development of multiple features

```bash
/dev Build an auth system.
# (pause auth development)
/dev Build a payment system.
# → Detects file conflicts with existing PLAN_auth → User chooses
```

### Using individual skills

```bash
# Create and review a plan
/feature-planner

# Add phases to an existing plan
/feature-planner update PLAN_auth

# Verify current phase only
/verify-implementation --current-phase

# Check skill coverage
/manage-skills
```

## Error Handling

### Automatic Blocker Reporting

After 3 consecutive build/test failures, a blocker report is presented with options:

1. **Keep trying** — Fix with a different approach
2. **Skip phase** — Move to the next phase
3. **Abort** — Save current state and resume later with `/dev PLAN_<name>`

### Subagent Failures

| Path | Strategy |
|------|----------|
| Critical path (feature-planner PLAN creation) | Report to user and abort on failure |
| Parallel execution (skill-writer, test-runner) | Keep successful results, retry failures sequentially once |
| Non-critical path (code-reviewer) | Retry once, skip and continue on second failure |
| Fallback available (codebase-scanner) | Collect minimal context via `git diff`, `ls`, etc. and continue |

## Cross-Skill Recommendations

Each skill automatically recommends other skills based on execution results:

- `verify-implementation` → Recommends `/manage-skills` when uncovered files are found
- `manage-skills` → Recommends `/verify-implementation` after creating/updating skills
- No skills or plans exist → Recommends `/feature-planner`

## License

MIT
