<!-- JASP Shared Rules Auto-Setup -->
<!-- DO NOT REMOVE: This block ensures shared AI rules are available for all developers -->

> **Auto-setup check:** At the start of every conversation, verify that `.claude/shared-rules.md` exists and is a valid symlink (not broken). If it is missing or broken:
>
> 1. Find JASP-Shared: `find ~ -maxdepth 4 -type d -name "JASP-Shared" 2>/dev/null | head -5`
> 2. If not found, clone it as a sibling of this project: `git clone https://github.com/JASP-eu/JASP-Shared.git ../JASP-Shared`
> 3. Pull latest: `cd <SHARED_DIR> && git pull && cd -`
> 4. Create shared rules symlink:
>    - `mkdir -p .claude && ln -sf <SHARED_DIR>/AI-Rules/CLAUDE.md .claude/shared-rules.md`
> 5. Install agents globally (if not already present):
>    - `mkdir -p ~/.claude/agents`
>    - `for f in <SHARED_DIR>/AI-Rules/agents/*.md; do ln -sf "$f" ~/.claude/agents/$(basename "$f"); done`
> 6. Clean up project-level agent symlinks (if any exist from older setup):
>    - `for f in <SHARED_DIR>/AI-Rules/agents/*.md; do rm -f ".claude/agents/$(basename "$f")"; done`
>    - `rmdir .claude/agents 2>/dev/null || true`
> 7. Tell the user: "Shared AI rules were missing — I've set them up automatically."
>
> If `.claude/shared-rules.md` exists and is valid, skip this silently — do not mention it.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Auto Claude is a multi-agent autonomous coding framework that builds software through coordinated AI agent sessions. It uses the Claude Code SDK to run agents in isolated workspaces with security controls.

## Commands

### Setup
```bash
# Install dependencies (from auto-claude/)
uv venv && uv pip install -r requirements.txt
# Or: python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt

# Set up OAuth token
claude setup-token
# Add to auto-claude/.env: CLAUDE_CODE_OAUTH_TOKEN=your-token
```

### Creating and Running Specs
```bash
# Create a spec interactively
python auto-claude/spec_runner.py --interactive

# Create spec from task description
python auto-claude/spec_runner.py --task "Add user authentication"

# Force complexity level (simple/standard/complex)
python auto-claude/spec_runner.py --task "Fix button" --complexity simple

# Run autonomous build
python auto-claude/run.py --spec 001

# List all specs
python auto-claude/run.py --list
```

### Workspace Management
```bash
# Review changes in isolated worktree
python auto-claude/run.py --spec 001 --review

# Merge completed build into project
python auto-claude/run.py --spec 001 --merge

# Discard build
python auto-claude/run.py --spec 001 --discard
```

### QA Validation
```bash
# Run QA manually
python auto-claude/run.py --spec 001 --qa

# Check QA status
python auto-claude/run.py --spec 001 --qa-status
```

### Testing
```bash
# Install test dependencies (required first time)
cd auto-claude && uv pip install -r ../tests/requirements-test.txt

# Run all tests (use virtual environment pytest)
auto-claude/.venv/bin/pytest tests/ -v

# Run single test file
auto-claude/.venv/bin/pytest tests/test_security.py -v

# Run specific test
auto-claude/.venv/bin/pytest tests/test_security.py::test_bash_command_validation -v

# Skip slow tests
auto-claude/.venv/bin/pytest tests/ -m "not slow"
```

### Spec Validation
```bash
python auto-claude/validate_spec.py --spec-dir auto-claude/specs/001-feature --checkpoint all
```

### Releases
```bash
# Automated version bump and release (recommended)
node scripts/bump-version.js patch   # 2.5.5 -> 2.5.6
node scripts/bump-version.js minor   # 2.5.5 -> 2.6.0
node scripts/bump-version.js major   # 2.5.5 -> 3.0.0
node scripts/bump-version.js 2.6.0   # Set specific version

# Then push to trigger GitHub release workflows
git push origin main
git push origin v2.6.0
```

See [RELEASE.md](RELEASE.md) for detailed release process documentation.

## Architecture

### Core Pipeline

**Spec Creation (spec_runner.py)** - Dynamic 3-8 phase pipeline based on task complexity:
- SIMPLE (3 phases): Discovery → Quick Spec → Validate
- STANDARD (6-7 phases): Discovery → Requirements → [Research] → Context → Spec → Plan → Validate
- COMPLEX (8 phases): Full pipeline with Research and Self-Critique phases

**Implementation (run.py → agent.py)** - Multi-session build:
1. Planner Agent creates subtask-based implementation plan
2. Coder Agent implements subtasks (can spawn subagents for parallel work)
3. QA Reviewer validates acceptance criteria
4. QA Fixer resolves issues in a loop

### Key Components

- **client.py** - Claude SDK client with security hooks and tool permissions
- **security.py** + **project_analyzer.py** - Dynamic command allowlisting based on detected project stack
- **worktree.py** - Git worktree isolation for safe feature development
- **memory.py** - File-based session memory (primary, always-available storage)
- **graphiti_memory.py** - Optional graph-based cross-session memory with semantic search
- **graphiti_providers.py** - Multi-provider factory for Graphiti (OpenAI, Anthropic, Azure, Ollama, Google AI)
- **graphiti_config.py** - Configuration and validation for Graphiti integration
- **linear_updater.py** - Optional Linear integration for progress tracking

### Agent Prompts (auto-claude/prompts/)

| Prompt | Purpose |
|--------|---------|
| planner.md | Creates implementation plan with subtasks |
| coder.md | Implements individual subtasks |
| coder_recovery.md | Recovers from stuck/failed subtasks |
| qa_reviewer.md | Validates acceptance criteria |
| qa_fixer.md | Fixes QA-reported issues |
| spec_gatherer.md | Collects user requirements |
| spec_researcher.md | Validates external integrations |
| spec_writer.md | Creates spec.md document |
| spec_critic.md | Self-critique using ultrathink |
| complexity_assessor.md | AI-based complexity assessment |

### Spec Directory Structure

Each spec in `auto-claude/specs/XXX-name/` contains:
- `spec.md` - Feature specification
- `requirements.json` - Structured user requirements
- `context.json` - Discovered codebase context
- `implementation_plan.json` - Subtask-based plan with status tracking
- `qa_report.md` - QA validation results
- `QA_FIX_REQUEST.md` - Issues to fix (when rejected)

### Branching & Worktree Strategy

Auto Claude uses git worktrees for isolated builds. All branches stay LOCAL until user explicitly pushes:

```
main (user's branch)
└── auto-claude/{spec-name}  ← spec branch (isolated worktree)
```

**Key principles:**
- ONE branch per spec (`auto-claude/{spec-name}`)
- Parallel work uses subagents (agent decides when to spawn)
- NO automatic pushes to GitHub - user controls when to push
- User reviews in spec worktree (`.worktrees/{spec-name}/`)
- Final merge: spec branch → main (after user approval)

**Workflow:**
1. Build runs in isolated worktree on spec branch
2. Agent implements subtasks (can spawn subagents for parallel work)
3. User tests feature in `.worktrees/{spec-name}/`
4. User runs `--merge` to add to their project
5. User pushes to remote when ready

### Security Model

Three-layer defense:
1. **OS Sandbox** - Bash command isolation
2. **Filesystem Permissions** - Operations restricted to project directory
3. **Command Allowlist** - Dynamic allowlist from project analysis (security.py + project_analyzer.py)

Security profile cached in `.auto-claude-security.json`.

### Memory System

Dual-layer memory architecture:

**File-Based Memory (Primary)** - `memory.py`
- Zero dependencies, always available
- Human-readable files in `specs/XXX/memory/`
- Session insights, patterns, gotchas, codebase map

**Graphiti Memory (Optional Enhancement)** - `graphiti_memory.py`
- Graph database with semantic search (FalkorDB)
- Cross-session context retrieval
- Multi-provider support (V2):
  - LLM: OpenAI, Anthropic, Azure OpenAI, Ollama, Google AI (Gemini)
  - Embedders: OpenAI, Voyage AI, Azure OpenAI, Ollama, Google AI

Enable with: `GRAPHITI_ENABLED=true` + provider credentials. See `.env.example`.

## Project Structure

Auto Claude can be used in two ways:

**As a standalone CLI tool** (original project):
```bash
python auto-claude/run.py --spec 001
```

**With the optional Electron frontend** (`auto-claude-ui/`):
- Provides a GUI for task management and progress tracking
- Wraps the CLI commands - the backend works independently

**Directory layout:**
- `auto-claude/` - Python backend/CLI (the framework code)
- `auto-claude-ui/` - Optional Electron frontend
- `.auto-claude/specs/` - Per-project data (specs, plans, QA reports) - gitignored

## Skills & Agents

This project uses a **hybrid approach**: domain **Skills** load guidelines on-demand (before coding), and **Agents** verify code (after coding).

### Skills (load before coding)

Load a domain skill to internalize patterns before writing code:

| Skill              | When to load                                           |
| ------------------ | ------------------------------------------------------ |
| `/branding`        | Before using brand colors, logos, icons, or typography |
| `/backend`         | Before writing API routes, DB, server code             |
| `/frontend`        | Before writing React components, UI, styling           |
| `/quality`         | Before self-reviewing code                             |
| `/testing`         | Before writing tests                                   |
| `/security`        | Before writing auth, secrets, input handling code      |
| `/devops`          | Before git ops, CI/CD, Docker, deploys                 |
| `/design-review`   | Before multi-agent implementation or after failures    |
| `/session-summary` | At end of session or before handoff                    |
| `/implement-issue` | To implement a GitHub issue end-to-end                 |

### Agents (verify after coding)

Agents are **mandatory** quality gates. Spawn them after implementation:

| Trigger                                              | Agent(s)                                      | Purpose                                                        |
| ---------------------------------------------------- | --------------------------------------------- | -------------------------------------------------------------- |
| After writing or modifying any code                  | `quality`                                     | Lint, type check, build verification                           |
| After writing or modifying tests                     | `testing`                                     | Run tests, verify coverage                                     |
| When modifying API routes, DB, or server code        | `backend` + `quality`                         | Backend verification                                           |
| When modifying UI components, styles, or client code | `frontend` + `quality`                        | Frontend verification                                          |
| When touching auth, secrets, input handling, or deps | `security`                                    | Security audit                                                 |
| When modifying CI/CD, Docker, or infra config        | `devops`                                      | Git workflow & infrastructure                                  |
| Before creating a PR                                 | `quality` + `pr-review-toolkit:code-reviewer` | Review against standards                                       |
| When implementing a GitHub issue                     | Load `/implement-issue` skill                 | Full lifecycle: plan → implement → PR → Copilot review → fix   |
| After decisions are made or at session end           | `scribe`                                      | Consolidate decisions.md, deduplicate, maintain knowledge base |
| Continuously during team execution                   | `watchdog`                                    | Monitor agent health, detect stalls, enforce budgets           |
| After performance-sensitive changes                  | `performance`                                 | Profile, benchmark, detect regressions                         |

**Never skip an agent.** If an agent reports issues, fix them before continuing.

### Mandatory Workflow (NEVER skip any step)

Every feature, bugfix, or refactor **must** follow this exact sequence. Do not wait for the user to remind you — execute each step automatically:

```
1. BEFORE coding  → Load relevant Skill (/backend, /frontend, etc.)
2. DURING coding  → Apply loaded patterns and conventions
3. AFTER coding   → Spawn quality + domain Agents, fix all reported issues
4. CREATE branch  → Feature branch (feat/, fix/, refactor/) — never commit to main
5. COMMIT         → Conventional commit message
6. CREATE PR      → Open PR with summary, link to issue if applicable
7. BEFORE PR      → Spawn quality + pr-review-toolkit:code-reviewer agents
```

**Every non-trivial task gets a GitHub issue.** Create one if none exists. Link PRs to issues. Update issues with decisions and status.

### Agent Teams

When the Teams feature is enabled (TeamCreate / SendMessage tools are available), **use agent teams for multi-step or cross-concern tasks**. Instead of running agents sequentially yourself, spawn a team and let agents work in parallel.

**Use teams when:**

- A task touches multiple concerns (e.g., backend + frontend + tests) — spawn agents for each in parallel
- A feature requires research, implementation, and review — run exploration and coding concurrently
- Multiple independent files or modules need changes — parallelize the work

**How to use teams:**

1. Create a team with `TeamCreate`
2. Break the work into tasks with `TaskCreate`
3. Spawn teammates via `Task` tool with `team_name` and appropriate `subagent_type`
4. Coordinate via `TaskList` / `SendMessage`
5. Shut down the team when done

**If Teams is not available**, fall back to sequential agent delegation as described in the table above.

### Task Lists

For any non-trivial task (3+ steps, multi-file changes, or multi-stage work), **create a task list upfront** using `TaskCreate`. This makes progress visible, prevents steps from being forgotten, and helps recover context if the conversation gets long.

**Always use task lists when:**

- Implementing a feature that spans multiple files or components
- Performing a multi-step refactor or migration
- Working through a bug that requires investigation, fix, and verification
- Any request where the user provides multiple items to complete

**Task list workflow:**

1. Break the work into discrete, actionable tasks with `TaskCreate`
2. Mark each task `in_progress` before starting it
3. Mark each task `completed` only when fully done (tests pass, no errors)
4. If a task is blocked or reveals new work, create follow-up tasks
5. Check `TaskList` after completing each task to pick up the next one
