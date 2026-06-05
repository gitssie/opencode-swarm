# OpenCode Swarm — Multi-Agent Collaborative Development Setup Guide

> One-click setup for architect + coder + designer agents + CodeGraph semantic index + Beads task graph + Muninn persistent memory — a complete AI coding workflow.

---

## Architecture Overview

```
┌────────────────────────────────────────────────────┐
│                    OpenCode CLI                      │
├────────────────────────────────────────────────────┤
│  Architect (primary)                                │
│  ├─ Plans requirements, decomposes tasks,           │
│  │  delegates to Coder/Designer                    │
│  ├─ Uses Beads (bd) to manage task dependency graph  │
│  └─ Uses Muninn to persist architectural decisions   │
│                                                     │
│  Coder (subagent)                                   │
│  ├─ Implements business logic, tests, refactoring    │
│  ├─ Uses CodeGraph to understand the codebase       │
│  └─ Follows TDD workflow                            │
│                                                     │
│  Designer (subagent)                                │
│  ├─ Produces accessible, responsive UI scaffolding   │
│  └─ Provides TODO-stubbed component structure       │
│       for Coder                                     │
├────────────────────────────────────────────────────┤
│  MCP Integrations                                   │
│  ├─ CodeGraph MCP — Semantic code index             │
│  ├─ Beads MCP — Graph-structured task tracking      │
│  └─ Muninn MCP — Cross-session persistent memory    │
└────────────────────────────────────────────────────┘
```

---

## Step 0: Pre-Installation Check

Before installing anything, check which components you already have:

```bash
# Check what's already installed
opencode --version     2>/dev/null && echo "✓ OpenCode" || echo "✗ OpenCode — install in Step 1"
uv --version           2>/dev/null && echo "✓ uv (Python)" || echo "✗ uv — install in Step 1"
codegraph version      2>/dev/null && echo "✓ CodeGraph" || echo "✗ CodeGraph — install in Step 2"
bd --version           2>/dev/null && echo "✓ Beads" || echo "✗ Beads — install in Step 3"
```

| Component | Already Installed? | Action |
|-----------|-------------------|--------|
| **OpenCode** | Run `opencode --version` | Skip Step 1 |
| **uv (Python)** | Run `uv --version` | Skip Step 1's uv section |
| **CodeGraph** | Run `codegraph version` | Skip Step 2 install; run `codegraph init -i` per project |
| **Beads** | Run `bd --version` | Skip Step 3 install; run `bd init` + `bd setup opencode` per project |
| **Muninn Memory** | Do you have a remote Muninn instance running? | If yes: skip Step 4 install, go directly to [Step 4.2](#42-configure-in-opencodejson) to configure the URL. If no: proceed with Step 4 to deploy locally. |

> **Tip**: If Muninn is already deployed remotely (shared team instance, cloud hosting, etc.), you only need the URL and token — skip the Docker/source install in Step 4 altogether.

---

## Step 1: Prerequisites

| Dependency | Version | Notes |
|------------|---------|-------|
| **OpenCode** | latest | `npm install -g opencode` |
| **Node.js** | >=18 | CodeGraph bundles its own runtime, Node optional for install |
| **Python + uv** | >=3.10 | Required for Muninn MCP and Beads MCP |
| **Git** | >=2.0 | Version control |

```bash
# Install opencode
npm install -g opencode

# Install uv (Python package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Verify all tools
opencode --version
uv --version
```

---

## Step 2: Install CodeGraph (Semantic Code Index)

CodeGraph builds a pre-indexed knowledge graph for your project. All agents access it via the MCP protocol, avoiding grep/read traversal.

```bash
# Install globally
npm install -g @colbymchenry/codegraph
```

Then initialize the index in each project:

```bash
cd /path/to/your-project
codegraph init -i
```

This creates `.codegraph/` in the project root. Then configure CodeGraph as an MCP server manually in your opencode config (see Step 5.2).

```bash
# After initialization, restart opencode for the MCP server to take effect

---

## Step 3: Install Beads (Task Graph Management)

Beads (`bd`) = a distributed, graph-structured task tracker built for AI agents. Architect uses it to create, assign, and manage task dependencies.

```bash
# Option 1: Homebrew (recommended)
brew install beads

# Option 2: One-liner script
curl -fsSL https://raw.githubusercontent.com/gastownhall/beads/main/scripts/install.sh | bash

# Option 3: npm
npm install -g @beads/bd
```

### Initialize in a project

```bash
cd /path/to/your-project
bd init
```

### Inject instructions for OpenCode

```bash
bd setup opencode
```

This writes a Beads workflow instruction block into the project `AGENTS.md`. Architect reads it and uses `bd create`, `bd show`, `bd close`, etc. to manage tasks.

### Quick command reference

```bash
bd ready              # List unblocked, pending tasks
bd create "Title" -t feature -p 1    # Create task (-t: bug/feature/task, -p: 0-3)
bd show bd-42 --json  # View task details
bd update bd-42 --claim -q   # Claim a task
bd close bd-42 --reason "Done" -q  # Close a task
bd prime              # Print current workflow context
```

---

## Step 4: Install Muninn Memory (Cross-Session Persistent Memory)

Muninn is a persistent memory database. Agents use it to save architectural decisions, coding conventions, and known pitfalls across sessions.

> **Decision required**: Do you already have a remote Muninn instance running (team server, cloud, etc.)? If yes, skip to [Step 4.2](#42-configure-in-opencodejson) and just configure the URL. If no, choose a local deployment method below.

### 4.1 Start the Muninn service

Choose **one** deployment method:

**Option A: Docker (recommended)**
```bash
docker run -d --name muninn \
  -p 8750:8750 \
  -v muninn_data:/data \
  ghcr.io/your-org/muninn:latest
```

**Option B: From source (requires Go)**
```bash
git clone https://github.com/your-org/muninn.git
cd muninn
go run ./cmd/server
```

The service listens on `http://127.0.0.1:8750`.

> **Tip**: Docker is easier to set up and isolates the service. Source builds give you full control but require a Go toolchain.

### 4.2 Configure in opencode.json

```json
{
  "mcp": {
    "memory": {
      "type": "remote",
      "url": "http://127.0.0.1:8750/mcp",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer <your-muninn-token>"
      }
    }
  }
}
```

---

## Step 5: Configure the OpenCode Environment

OpenCode supports two placement strategies: **global** (shared across all projects) and **project-level** (scoped to one project). Project-level is recommended — it keeps config, agents, and behavior specs versioned alongside your code.

| Resource | Global Location | Project Location | Recommendation |
|----------|----------------|-----------------|----------------|
| Config (`opencode.json`) | `~/.config/opencode/opencode.json` | `.opencode/opencode.json` | **Project-level** |
| Agents (`architect.md`, etc.) | `~/.config/opencode/agents/` | `.opencode/agents/` | Your choice |
| Behavior spec (`AGENTS.md`) | `~/.config/opencode/AGENTS.md` | `.opencode/AGENTS.md` | Your choice |

> **Your choice**: Before proceeding, decide where to place each resource. The steps below show both paths — pick one and skip the other.

### 5.0 Required environment variable

OpenCode's multi-agent background subagent feature requires an experimental flag:

```bash
export OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true
```

Add this to your shell profile (`~/.zshrc`, `~/.bashrc`, etc.):

```bash
echo 'export OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS=true' >> ~/.zshrc
```

### 5.1 Agent definition files

Copy the three agent files. Pick **one** location:

```bash
# Option A: Global (shared across all projects)
mkdir -p ~/.config/opencode/agents
cp agents/architect.md ~/.config/opencode/agents/
cp agents/coder.md ~/.config/opencode/agents/
cp agents/designer.md ~/.config/opencode/agents/

# Option B: Project-level (recommended for single-project repos)
mkdir -p .opencode/agents
cp agents/architect.md .opencode/agents/
cp agents/coder.md .opencode/agents/
cp agents/designer.md .opencode/agents/
```

**Agent role descriptions**:

| Agent | Type | Responsibility |
|-------|------|----------------|
| **architect** | `primary` | Understand requirements → explore code → decompose into bd tasks → dispatch asynchronously to coder/designer |
| **coder** | `subagent` | Pull bd tasks → load TDD + grill-me skills → implement code → verify → commit |
| **designer** | `subagent` | Pull UI design tasks → load frontend-design + grill-me skills → produce accessible, responsive scaffolding |

### 5.2 Configuration file (`opencode.json`)

**Recommendation: use project-level** so your config is versioned with the code and doesn't pollute global settings.

Pick **one** location:

```bash
# Option A: Project-level (recommended)
mkdir -p .opencode
printf '{\n  "$schema": "https://opencode.ai/config.json",\n  "model": "deepseek/deepseek-v4-pro",\n  "small_model": "deepseek/deepseek-v4-flash",\n  "agent": {\n    "general": {\n      "model": "deepseek/deepseek-v4-pro"\n    }\n  },\n  "mcp": {\n    "lsp": {\n      "type": "local",\n      "command": ["codegraph", "serve", "--mcp"],\n      "enabled": true\n    },\n    "memory": {\n      "type": "remote",\n      "url": "http://127.0.0.1:8750/mcp",\n      "enabled": true,\n      "headers": {\n        "Authorization": "Bearer <your-muninn-token>"\n      }\n    }\n  }\n}\n' > .opencode/opencode.json

# Option B: Global
printf '{\n  "$schema": "https://opencode.ai/config.json",\n  "model": "deepseek/deepseek-v4-pro",\n  "small_model": "deepseek/deepseek-v4-flash",\n  "agent": {\n    "general": {\n      "model": "deepseek/deepseek-v4-pro"\n    }\n  },\n  "mcp": {\n    "lsp": {\n      "type": "local",\n      "command": ["codegraph", "serve", "--mcp"],\n      "enabled": true\n    },\n    "memory": {\n      "type": "remote",\n      "url": "http://127.0.0.1:8750/mcp",\n      "enabled": true,\n      "headers": {\n        "Authorization": "Bearer <your-muninn-token>"\n      }\n    }\n  }\n}\n' > ~/.config/opencode/opencode.json
```

> **Notes**:
> - You must add `lsp` (CodeGraph) and `memory` (Muninn) MCP config manually.
> - Replace `<your-muninn-token>` with your actual Muninn token, or remove the `headers` block if no auth is required.

### 5.3 Behavior specification (`AGENTS.md`)

`AGENTS.md` is injected as a prefix into every conversation. It contains CodeGraph tool rules, workflow state machine, Muninn memory protocol, and TDD discipline.

> **Decision required**: Do you want agents to follow this behavior specification? If not, skip this step entirely.

If yes, pick the same location you chose for agents (5.1):

```bash
# Option A: Global
cp AGENTS.md ~/.config/opencode/AGENTS.md

# Option B: Project-level
cp AGENTS.md .opencode/AGENTS.md
```

> **Note**: `bd setup opencode` also writes instructions into the project root `AGENTS.md`. This is separate — keep the swarm behavior spec at your chosen level and let `bd` write its workflow instructions at the project root.

---

## Step 7: Initialize Your Project

Now that config and agents are placed (Step 5), initialize the remaining tooling per project:

```bash
cd /path/to/my-project

# 1. Initialize CodeGraph semantic index
codegraph init -i

# 2. Initialize Beads task tracking
bd init
bd setup opencode

# 3. (Optional) Remind agents to use Beads + CodeGraph in project AGENTS.md
echo '## Issue Tracking
Use `bd` for task tracking: `bd ready` for unblocked work, `bd create`, `bd close`.
Run `bd prime` at session start for workflow context.

## CodeGraph
This project has CodeGraph initialized (.codegraph/ exists).
Use `lsp_codegraph_explore` as your PRIMARY code exploration tool.' >> AGENTS.md
```

---

## Step 8: Launch and Use

```bash
cd /path/to/my-project
opencode
```

Once inside the opencode interactive interface, Architect takes over as the primary agent. You can start like this:

```
Implement user login with email+password and Google OAuth
```

Architect will:
1. Call `memory_muninn_recall` to check prior architectural decisions
2. Use CodeGraph to explore the codebase
3. Use `bd create` to decompose requirements into tasks
4. Dispatch them asynchronously to coder and designer

---

## Appendix A: Skills

Skills change how agents work. Install each one via CLI:

```bash
npx skills add https://github.com/obra/superpowers --skill <skill-name> -a opencode
```

| Skill | Used By | Command | Effect |
|-------|---------|---------|--------|
| `tdd` | coder | `npx skills add ... --skill tdd -a opencode` | Enforces red-green-refactor cycle; no tests = not done |
| `grill-me` | coder, designer | `npx skills add ... --skill grill-me -a opencode` | Allows agents to ask architect follow-up questions instead of guessing |
| `frontend-design` | designer | `npx skills add ... --skill frontend-design -a opencode` | Injects design spec checklist, accessibility checks, responsive strategy |
| `caveman` | (on-demand) | `npx skills add ... --skill caveman -a opencode` | Ultra-compressed conversation, saves ~75% tokens |
| `diagnose` | (on-demand) | `npx skills add ... --skill diagnose -a opencode` | Systematic debugging: reproduce → minimize → hypothesize → instrument → fix → regress |
| `vue` | (on-demand) | `npx skills add ... --skill vue -a opencode` | Vue 3 Composition API specialization |
| `quasar-skilld` | (on-demand) | `npx skills add ... --skill quasar-skilld -a opencode` | Quasar UI framework specialization |

---

## Appendix B: Complete Directory Reference

```
~/.config/opencode/           # Global (if you chose global in Step 5)
├── opencode.json             # Global config
├── AGENTS.md                 # Global agent behavior spec
├── agents/
│   ├── architect.md          # Orchestrator agent, primary
│   ├── coder.md              # Code implementation agent, subagent
│   └── designer.md           # UI design agent, subagent
├── commands/                 # Custom slash commands
├── skills/                   # Skill files
├── plugins/                  # Plugin files
└── opencode-swarm.md         # This document

Project directory/
├── .opencode/
│   ├── opencode.json         # Project config (recommended)
│   ├── AGENTS.md             # (Optional) Project-level behavior spec
│   └── agents/               # (Optional) Project-level agents
│       ├── architect.md
│       ├── coder.md
│       └── designer.md
├── .codegraph/               # CodeGraph index (codegraph init -i)
├── .beads/                   # Beads database (bd init)
└── AGENTS.md                 # Beads workflow instructions (bd setup opencode)
```

---

## Appendix C: FAQ

### Q: CodeGraph MCP reports "not initialized"
```bash
cd your-project && codegraph init -i
```

### Q: bd command not found
```bash
# Homebrew
brew install beads

# Or check PATH
export PATH="$HOME/.local/bin:$PATH"
```

### Q: Muninn memory connection fails
Ensure the Muninn container/service is running:
```bash
docker ps | grep muninn
curl http://127.0.0.1:8750/mcp
```

### Q: Architect doesn't dispatch tasks to Coder
- Check that `agents/architect.md` has `mode: primary`
- Check that `agents/coder.md` has `mode: subagent`
- Ensure your opencode version supports multi-agent mode

### Q: How to verify the full environment is ready
```bash
# 1. Check each component
codegraph status .    # CodeGraph index status
bd ready              # Any pending Beads tasks
opencode --version    # OpenCode version

# 2. Inside an opencode session
# Architect should automatically: memory_muninn_recall → read AGENTS.md
# Subsequent exploration should use lsp_codegraph_* tools
```
