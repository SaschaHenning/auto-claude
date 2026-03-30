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

Auto Claude is a multi-agent autonomous coding framework that builds software through coordinated AI agent sessions. It uses the Claude Agent SDK to run agents in isolated workspaces with security controls.

**CRITICAL: All AI interactions use the Claude Agent SDK (`claude-agent-sdk` package), NOT the Anthropic API directly.**

## Project Structure

```
autonomous-coding/
├── apps/
│   ├── backend/           # Python backend/CLI - ALL agent logic lives here
│   │   ├── core/          # Client, auth, security
│   │   ├── agents/        # Agent implementations
│   │   ├── spec_agents/   # Spec creation agents
│   │   ├── integrations/  # Graphiti, Linear, GitHub
│   │   └── prompts/       # Agent system prompts
│   └── frontend/          # Electron desktop UI
├── guides/                # Documentation
├── tests/                 # Test suite
└── scripts/               # Build and utility scripts
```

**When working with AI/LLM code:**
- Look in `apps/backend/core/client.py` for the Claude SDK client setup
- Reference `apps/backend/agents/` for working agent implementations
- Check `apps/backend/spec_agents/` for spec creation agent examples
- NEVER use `anthropic.Anthropic()` directly - always use `create_client()` from `core.client`

**Frontend (Electron Desktop App):**
- Built with Electron, React, TypeScript
- AI agents can perform E2E testing using the Electron MCP server
- When bug fixing or implementing features, use the Electron MCP server for automated testing
- See "End-to-End Testing" section below for details

## Commands

### Setup

**Requirements:**
- Python 3.12+ (required for backend)
- Node.js (for frontend)

```bash
# Install all dependencies from root
npm run install:all

# Or install separately:
# Backend (from apps/backend/)
cd apps/backend && uv venv && uv pip install -r requirements.txt

# Frontend (from apps/frontend/)
cd apps/frontend && npm install

# Authenticate (token auto-saved to Keychain)
claude
# Then type: /login
# Press Enter to open browser and complete OAuth
```

### Creating and Running Specs
```bash
cd apps/backend

# Create a spec interactively
python spec_runner.py --interactive

# Create spec from task description
python spec_runner.py --task "Add user authentication"

# Force complexity level (simple/standard/complex)
python spec_runner.py --task "Fix button" --complexity simple

# Run autonomous build
python run.py --spec 001

# List all specs
python run.py --list
```

### Workspace Management
```bash
cd apps/backend

# Review changes in isolated worktree
python run.py --spec 001 --review

# Merge completed build into project
python run.py --spec 001 --merge

# Discard build
python run.py --spec 001 --discard
```

### QA Validation
```bash
cd apps/backend

# Run QA manually
python run.py --spec 001 --qa

# Check QA status
python run.py --spec 001 --qa-status
```

### Testing
```bash
# Install test dependencies (required first time)
cd apps/backend && uv pip install -r ../../tests/requirements-test.txt

# Run all tests (use virtual environment pytest)
apps/backend/.venv/bin/pytest tests/ -v

# Run single test file
apps/backend/.venv/bin/pytest tests/test_security.py -v

# Run specific test
apps/backend/.venv/bin/pytest tests/test_security.py::test_bash_command_validation -v

# Skip slow tests
apps/backend/.venv/bin/pytest tests/ -m "not slow"

# Or from root
npm run test:backend
```

### Spec Validation
```bash
python apps/backend/validate_spec.py --spec-dir apps/backend/specs/001-feature --checkpoint all
```

### Releases
```bash
# 1. Bump version on your branch (creates commit, no tag)
node scripts/bump-version.js patch   # 2.8.0 -> 2.8.1
node scripts/bump-version.js minor   # 2.8.0 -> 2.9.0
node scripts/bump-version.js major   # 2.8.0 -> 3.0.0

# 2. Push and create PR to main
git push origin your-branch
gh pr create --base main

# 3. Merge PR → GitHub Actions automatically:
#    - Creates tag
#    - Builds all platforms
#    - Creates release with changelog
#    - Updates README
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
3. QA Reviewer validates acceptance criteria (can perform E2E testing via Electron MCP for frontend changes)
4. QA Fixer resolves issues in a loop (with E2E testing to verify fixes)

### Key Components (apps/backend/)

**Core Infrastructure:**
- **core/client.py** - Claude Agent SDK client factory with security hooks and tool permissions
- **core/security.py** - Dynamic command allowlisting based on detected project stack
- **core/auth.py** - OAuth token management for Claude SDK authentication
- **agents/** - Agent implementations (planner, coder, qa_reviewer, qa_fixer)
- **spec_agents/** - Spec creation agents (gatherer, researcher, writer, critic)

**Memory & Context:**
- **integrations/graphiti/** - Graphiti memory system (mandatory)
  - `queries_pkg/graphiti.py` - Main GraphitiMemory class
  - `queries_pkg/client.py` - LadybugDB client wrapper
  - `queries_pkg/queries.py` - Graph query operations
  - `queries_pkg/search.py` - Semantic search logic
  - `queries_pkg/schema.py` - Graph schema definitions
- **graphiti_config.py** - Configuration and validation for Graphiti integration
- **graphiti_providers.py** - Multi-provider factory (OpenAI, Anthropic, Azure, Ollama, Google AI)
- **agents/memory_manager.py** - Session memory orchestration

**Workspace & Security:**
- **cli/worktree.py** - Git worktree isolation for safe feature development
- **context/project_analyzer.py** - Project stack detection for dynamic tooling
- **auto_claude_tools.py** - Custom MCP tools integration

**Integrations:**
- **linear_updater.py** - Optional Linear integration for progress tracking
- **runners/github/** - GitHub Issues & PRs automation
- **Electron MCP** - E2E testing integration for QA agents (Chrome DevTools Protocol)
  - Enabled with `ELECTRON_MCP_ENABLED=true` in `.env`
  - Allows QA agents to interact with running Electron app
  - See "End-to-End Testing" section for details

### Agent Prompts (apps/backend/prompts/)

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

Each spec in `.auto-claude/specs/XXX-name/` contains:
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

### Contributing to Upstream

**CRITICAL: When submitting PRs to AndyMik90/Auto-Claude, always target the `develop` branch, NOT `main`.**

**Correct workflow for contributions:**
1. Fetch upstream: `git fetch upstream`
2. Create feature branch from upstream/develop: `git checkout -b fix/my-fix upstream/develop`
3. Make changes and commit with sign-off: `git commit -s -m "fix: description"`
4. Push to your fork: `git push origin fix/my-fix`
5. Create PR targeting `develop`: `gh pr create --repo AndyMik90/Auto-Claude --base develop`

**Verify before PR:**
```bash
# Ensure only your commits are included
git log --oneline upstream/develop..HEAD
```

### Security Model

Three-layer defense:
1. **OS Sandbox** - Bash command isolation
2. **Filesystem Permissions** - Operations restricted to project directory
3. **Command Allowlist** - Dynamic allowlist from project analysis (security.py + project_analyzer.py)

Security profile cached in `.auto-claude-security.json`.

### Claude Agent SDK Integration

**CRITICAL: Auto Claude uses the Claude Agent SDK for ALL AI interactions. Never use the Anthropic API directly.**

**Client Location:** `apps/backend/core/client.py`

The `create_client()` function creates a configured `ClaudeSDKClient` instance with:
- Multi-layered security (sandbox, permissions, security hooks)
- Agent-specific tool permissions (planner, coder, qa_reviewer, qa_fixer)
- Dynamic MCP server integration based on project capabilities
- Extended thinking token budget control

**Example usage in agents:**
```python
from core.client import create_client

# Create SDK client (NOT raw Anthropic API client)
client = create_client(
    project_dir=project_dir,
    spec_dir=spec_dir,
    model="claude-sonnet-4-5-20250929",
    agent_type="coder",
    max_thinking_tokens=None  # or 5000/10000/16000
)

# Run agent session
response = client.create_agent_session(
    name="coder-agent-session",
    starting_message="Implement the authentication feature"
)
```

**Why use the SDK:**
- Pre-configured security (sandbox, allowlists, hooks)
- Automatic MCP server integration (Context7, Linear, Graphiti, Electron, Puppeteer)
- Tool permissions based on agent role
- Session management and recovery
- Unified API across all agent types

**Directory layout:**
- `auto-claude/` - Python backend/CLI (the framework code)
- `auto-claude-ui/` - Optional Electron frontend
- `.auto-claude/specs/` - Per-project data (specs, plans, QA reports) - gitignored
**Where to find working examples:**
- `apps/backend/agents/planner.py` - Planner agent
- `apps/backend/agents/coder.py` - Coder agent
- `apps/backend/agents/qa_reviewer.py` - QA reviewer
- `apps/backend/agents/qa_fixer.py` - QA fixer
- `apps/backend/spec_agents/` - Spec creation agents

### Memory System

**Graphiti Memory (Mandatory)** - `integrations/graphiti/`

Auto Claude uses Graphiti as its primary memory system with embedded LadybugDB (no Docker required):

- **Graph database with semantic search** - Knowledge graph for cross-session context
- **Session insights** - Patterns, gotchas, discoveries automatically extracted
- **Multi-provider support:**
  - LLM: OpenAI, Anthropic, Azure OpenAI, Ollama, Google AI (Gemini)
  - Embedders: OpenAI, Voyage AI, Azure OpenAI, Ollama, Google AI
- **Modular architecture:** (`integrations/graphiti/queries_pkg/`)
  - `graphiti.py` - Main GraphitiMemory class
  - `client.py` - LadybugDB client wrapper
  - `queries.py` - Graph query operations
  - `search.py` - Semantic search logic
  - `schema.py` - Graph schema definitions

**Configuration:**
- Set provider credentials in `apps/backend/.env` (see `.env.example`)
- Required env vars: `GRAPHITI_ENABLED=true`, `ANTHROPIC_API_KEY` or other provider keys
- Memory data stored in `.auto-claude/specs/XXX/graphiti/`

**Usage in agents:**
```python
from integrations.graphiti.memory import get_graphiti_memory

memory = get_graphiti_memory(spec_dir, project_dir)
context = memory.get_context_for_session("Implementing feature X")
memory.add_session_insight("Pattern: use React hooks for state")
```

## Development Guidelines

### Frontend Internationalization (i18n)

**CRITICAL: Always use i18n translation keys for all user-facing text in the frontend.**

The frontend uses `react-i18next` for internationalization. All labels, buttons, messages, and user-facing text MUST use translation keys.

**Translation file locations:**
- `apps/frontend/src/shared/i18n/locales/en/*.json` - English translations
- `apps/frontend/src/shared/i18n/locales/fr/*.json` - French translations

**Translation namespaces:**
- `common.json` - Shared labels, buttons, common terms
- `navigation.json` - Sidebar navigation items, sections
- `settings.json` - Settings page content
- `dialogs.json` - Dialog boxes and modals
- `tasks.json` - Task/spec related content
- `errors.json` - Error messages (structured error information with substitution support)
- `onboarding.json` - Onboarding wizard content
- `welcome.json` - Welcome screen content

**Usage pattern:**
```tsx
import { useTranslation } from 'react-i18next';

// In component
const { t } = useTranslation(['navigation', 'common']);

// Use translation keys, NOT hardcoded strings
<span>{t('navigation:items.githubPRs')}</span>  // ✅ CORRECT
<span>GitHub PRs</span>                          // ❌ WRONG
```

**Error messages with substitution:**

```tsx
// For error messages with dynamic content, use interpolation
const { t } = useTranslation(['errors']);

// errors.json: { "task": { "parseError": "Failed to parse: {{error}}" } }
<span>{t('errors:task.parseError', { error: errorMessage })}</span>
```

**When adding new UI text:**
1. Add the translation key to ALL language files (at minimum: `en/*.json` and `fr/*.json`)
2. Use `namespace:section.key` format (e.g., `navigation:items.githubPRs`)
3. Never use hardcoded strings in JSX/TSX files

### Cross-Platform Development

**CRITICAL: This project supports Windows, macOS, and Linux. Platform-specific bugs are the #1 source of breakage.**

#### The Problem

When developers on macOS fix something using Mac-specific assumptions, it breaks on Windows. When Windows developers fix something, it breaks on macOS. This happens because:

1. **CI only tested on Linux** - Platform-specific bugs weren't caught until after merge
2. **Scattered platform checks** - `process.platform === 'win32'` checks were spread across 50+ files
3. **Hardcoded paths** - Direct paths like `C:\Program Files` or `/opt/homebrew/bin` throughout code

#### The Solution

**1. Centralized Platform Abstraction**

All platform-specific code now lives in dedicated modules:

- **Frontend:** `apps/frontend/src/main/platform/`
- **Backend:** `apps/backend/core/platform/`

**Import from these modules instead of checking `process.platform` directly:**

```typescript
// ❌ WRONG - Direct platform check
if (process.platform === 'win32') {
  // Windows logic
}

// ✅ CORRECT - Use abstraction
import { isWindows, getPathDelimiter } from './platform';

if (isWindows()) {
  // Windows logic
}
```

**2. Multi-Platform CI**

CI now tests on **all three platforms** (Windows, macOS, Linux). A PR cannot merge unless all platforms pass:

```yaml
# .github/workflows/ci.yml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
```

**3. Platform Module API**

The platform module provides:

| Function | Purpose |
|----------|---------|
| `isWindows()` / `isMacOS()` / `isLinux()` | OS detection |
| `getPathDelimiter()` | Get `;` (Windows) or `:` (Unix) |
| `getExecutableExtension()` | Get `.exe` (Windows) or `` (Unix) |
| `findExecutable(name)` | Find executables across platforms |
| `getBinaryDirectories()` | Get platform-specific bin paths |
| `requiresShell(command)` | Check if .cmd/.bat needs shell on Windows |

**4. Path Handling Best Practices**

```typescript
// ❌ WRONG - Hardcoded Windows path
const claudePath = 'C:\\Program Files\\Claude\\claude.exe';

// ❌ WRONG - Hardcoded macOS path
const brewPath = '/opt/homebrew/bin/python3';

// ❌ WRONG - Manual path joining
const fullPath = dir + '/subdir/file.txt';

// ✅ CORRECT - Use platform abstraction
import { findExecutable, joinPaths } from './platform';

const claudePath = await findExecutable('claude');
const fullPath = joinPaths(dir, 'subdir', 'file.txt');
```

**5. Testing Platform-Specific Code**

```typescript
// Mock process.platform for testing
import { isWindows } from './platform';

// In tests, use jest.mock or similar
jest.mock('./platform', () => ({
  isWindows: () => true  // Simulate Windows
}));
```

**6. When You Need Platform-Specific Code**

If you must write platform-specific code:

1. **Add it to the platform module** - Not scattered in your feature code
2. **Write tests for all platforms** - Mock `process.platform` to test each case
3. **Use feature detection** - Check for file/path existence, not just OS name
4. **Document why** - Explain the platform difference in comments

**7. Submitting Platform-Specific Fixes**

When fixing a platform-specific bug:

1. Ensure your fix doesn't break other platforms
2. Test locally if you have access to other OSs
3. Rely on CI to catch issues you can't test
4. Consider adding a test that mocks other platforms

**Example: Adding a New Tool Detection**

```typescript
// ✅ CORRECT - Add to platform/paths.ts
export function getMyToolPaths(): string[] {
  if (isWindows()) {
    return [
      joinPaths('C:', 'Program Files', 'MyTool', 'tool.exe'),
      // ... more Windows paths
    ];
  }
  return [
    joinPaths('/usr', 'local', 'bin', 'mytool'),
    // ... more Unix paths
  ];
}

// ✅ CORRECT - Use in your code
import { findExecutable, getMyToolPaths } from './platform';

const toolPath = await findExecutable('mytool', getMyToolPaths());
```

### End-to-End Testing (Electron App)

**IMPORTANT: When bug fixing or implementing new features in the frontend, AI agents can perform automated E2E testing using the Electron MCP server.**

The Electron MCP server allows QA agents to interact with the running Electron app via Chrome DevTools Protocol:

**Setup:**
1. Start the Electron app with remote debugging enabled:
   ```bash
   npm run dev  # Already configured with --remote-debugging-port=9222
   ```

2. Enable Electron MCP in `apps/backend/.env`:
   ```bash
   ELECTRON_MCP_ENABLED=true
   ELECTRON_DEBUG_PORT=9222  # Default port
   ```

**Available Testing Capabilities:**

QA agents (`qa_reviewer` and `qa_fixer`) automatically get access to Electron MCP tools:

1. **Window Management**
   - `mcp__electron__get_electron_window_info` - Get info about running windows
   - `mcp__electron__take_screenshot` - Capture screenshots for visual verification

2. **UI Interaction**
   - `mcp__electron__send_command_to_electron` with commands:
     - `click_by_text` - Click buttons/links by visible text
     - `click_by_selector` - Click elements by CSS selector
     - `fill_input` - Fill form fields by placeholder or selector
     - `select_option` - Select dropdown options
     - `send_keyboard_shortcut` - Send keyboard shortcuts (Enter, Ctrl+N, etc.)
     - `navigate_to_hash` - Navigate to hash routes (#settings, #create, etc.)

3. **Page Inspection**
   - `get_page_structure` - Get organized overview of page elements
   - `debug_elements` - Get debugging info about buttons and forms
   - `verify_form_state` - Check form state and validation
   - `eval` - Execute custom JavaScript code

4. **Logging**
   - `mcp__electron__read_electron_logs` - Read console logs for debugging

**Example E2E Test Flow:**

```python
# 1. Agent takes screenshot to see current state
agent: "Take a screenshot to see the current UI"
# Uses: mcp__electron__take_screenshot

# 2. Agent inspects page structure
agent: "Get page structure to find available buttons"
# Uses: mcp__electron__send_command_to_electron (command: "get_page_structure")

# 3. Agent clicks a button to navigate
agent: "Click the 'Create New Spec' button"
# Uses: mcp__electron__send_command_to_electron (command: "click_by_text", args: {text: "Create New Spec"})

# 4. Agent fills out a form
agent: "Fill the task description field"
# Uses: mcp__electron__send_command_to_electron (command: "fill_input", args: {placeholder: "Describe your task", value: "Add login feature"})

# 5. Agent submits and verifies
agent: "Click Submit and verify success"
# Uses: click_by_text → take_screenshot → verify result
```

**When to Use E2E Testing:**

- **Bug Fixes**: Reproduce the bug, apply fix, verify it's resolved
- **New Features**: Implement feature, test the UI flow end-to-end
- **UI Changes**: Verify visual changes and interactions work correctly
- **Form Validation**: Test form submission, validation, error handling

**Configuration in `core/client.py`:**

The client automatically enables Electron MCP tools for QA agents when:
- Project is detected as Electron (`is_electron` capability)
- `ELECTRON_MCP_ENABLED=true` is set
- Agent type is `qa_reviewer` or `qa_fixer`

**Note:** Screenshots are automatically compressed (1280x720, quality 60, JPEG) to stay under Claude SDK's 1MB JSON message buffer limit.

## Running the Application

**As a standalone CLI tool**:
```bash
cd apps/backend
python run.py --spec 001
```

**With the Electron frontend**:
```bash
npm start        # Build and run desktop app
npm run dev      # Run in development mode (includes --remote-debugging-port=9222 for E2E testing)
```

**For E2E Testing with QA Agents:**
1. Start the Electron app: `npm run dev`
2. Enable Electron MCP in `apps/backend/.env`: `ELECTRON_MCP_ENABLED=true`
3. Run QA: `python run.py --spec 001 --qa`
4. QA agents will automatically interact with the running app for testing

**Project data storage:**
- `.auto-claude/specs/` - Per-project data (specs, plans, QA reports, memory) - gitignored

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
