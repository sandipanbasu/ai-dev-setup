# RuFlo v3.5 — Universal MCP Integration Guide

> Covers: what problem this solves, installation across all four tools,
> SDLC phase usage, and key observations to keep in mind.
>
> Last updated: 2026-04-20 | RuFlo version: v3.5.2

---

## 1. What Problem Does This Solve?

### The core issue: AI coding tools are stateless and token-wasteful

Every session with OpenCode, Codex, Cursor, or Claude Code starts cold. The
agent has no memory of what worked last time, re-reads entire files to get
context, and has no way to coordinate with another agent running in a different
tool. This causes three compounding problems:

**Problem 1 — Token bloat**
Agents read whole files when they only need fragments. They repeat expensive
reasoning they've already done in a prior session. Context windows fill up
fast, especially on large codebases.

**Problem 2 — No shared intelligence across tools**
You use OpenCode for one task, Codex for another, Cursor for a third. Each
tool starts from scratch. Patterns, decisions, and learned conventions from one
session are invisible to all the others.

**Problem 3 — No task routing**
Every task — whether it's renaming a variable or redesigning an architecture —
gets routed to the same expensive model. There's no mechanism to say "this is
a $0 WASM transform, not a $0.015 Opus call."

### What RuFlo v3.5 fixes

RuFlo is an **agent orchestration layer** that runs as an MCP server. When
registered across all four tools, it provides:

| Problem | RuFlo solution |
|---|---|
| Token bloat from full file reads | `memory_search` retrieves cached patterns — agents read from memory, not files |
| No cross-session memory | SQLite + HNSW vector store on disk at `~/.claude/memory/` — persists forever |
| No shared intelligence across tools | One shared DB — learn in Cursor, recall in OpenCode |
| No task routing | 3-tier routing: WASM ($0) → Haiku/Sonnet ($0.0002–$0.003) → Opus ($0.015) |
| Agents work in isolation | Swarm coordination: parallel specialist agents with shared memory |

### What RuFlo does NOT fix

- It does not reduce tokens from your own messages/prompts
- It does not auto-fire optimisations in OpenCode, Codex, or Cursor (those
  hooks are Claude Code-specific — you call tools manually in other tools)
- It is not a CI/CD or production monitoring tool
- Memory savings only materialise after several sessions have built up
  the pattern store — day one value is limited to routing and swarm coordination

---

## 2. Architecture

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  OpenCode   │  │ Claude Code │  │   Codex     │  │   Cursor    │
│  (MCP via   │  │  (MCP via   │  │  (MCP via   │  │  (MCP via   │
│ jsonc file) │  │ claude mcp) │  │ config.toml)│  │  mcp.json)  │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │
       └────────────────┴────────────────┴────────────────┘
                                │
                   ┌────────────▼────────────┐
                   │   ruflo mcp start       │
                   │  (spawned per-tool as   │
                   │   a local subprocess)   │
                   └────────────┬────────────┘
                                │
                   ┌────────────▼────────────┐
                   │  ~/.claude/memory/      │
                   │  SQLite + HNSW index    │
                   │  (ONE shared DB across  │
                   │   all tools)            │
                   └─────────────────────────┘
```

Each tool spawns `ruflo mcp start` independently as a subprocess when the tool
launches. They all point to the same on-disk memory store — so patterns learned
in one tool are available in all others.

---

## 3. Installation

### Step 1 — Install ruflo globally (once)

```bash
npm install -g ruflo@latest
```

Global install eliminates cold-start latency (~20s on first `npx` run,
~3s cached). With global install, startup is instant.

Verify:
```bash
ruflo --version
# ruflo v3.5.2
```

### Step 2 — Initialise the memory store (once)

```bash
ruflo init --minimal
```

Creates `~/.claude/memory/` (SQLite + HNSW index). The `--minimal` flag skips
Claude Code-specific scaffolding (hooks, AGENTS.md) — we only want the shared
memory backend.

### Step 3 — Disable unused tool categories (recommended)

By default ruflo exposes **215 tools** across 20 categories. Each tool's name
+ description loads into the system prompt (~20 tokens each = ~4,300 tokens
overhead with all tools enabled). Trim this to what you actually need:

```bash
# Core set for memory + routing + agents (~34 tools, ~680 tokens overhead)
ruflo mcp toggle browser_* disable
ruflo mcp toggle github_* disable
ruflo mcp toggle terminal_* disable
ruflo mcp toggle neural_* disable
ruflo mcp toggle coordination_* disable
ruflo mcp toggle transfer_* disable
```

Recommended minimal enabled categories:

| Category | Tools | Purpose |
|---|---|---|
| Memory | 7 | `memory_search`, `memory_store`, `memory_retrieve` etc |
| Hooks | 10 | `hooks_route`, `hooks_model-route`, `hooks_metrics` etc |
| Agent | 7 | `agent_spawn`, `agent_list`, `agent_status` etc |
| Swarm | 4 | `swarm_init`, `swarm_status`, `swarm_health` etc |
| Config | 6 | `config_get`, `config_set` etc |
| **Total** | **~34** | **~680 tokens overhead** |

**Verify which tools are enabled after toggling:**

```bash
# See full list with enabled/disabled status
ruflo mcp tools

# Count how many are currently enabled
ruflo mcp tools | grep -c "Enabled"

# See only enabled tools
ruflo mcp tools | grep "Enabled"

# See tools by category
ruflo mcp tools | grep -A 20 "Memory"
```

### Step 4 — Register ruflo in each tool

---

#### OpenCode — `~/.config/opencode/opencode.jsonc`

**Current state:** `{}`

**Change to:**
```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "ruflo": {
      "type": "local",
      "command": "ruflo",
      "args": ["mcp", "start"],
      "enabled": true
    }
  }
}
```

---

#### Codex — `~/.codex/config.toml`

**Current state:** has stale `[mcp_servers.claude-flow]` entry (old package
name — same binary, but must be replaced, not duplicated — two entries =
two MCP processes spawned)

**Remove:**
```toml
[mcp_servers.claude-flow]
command = "npx"
args = ["claude-flow", "mcp", "start"]
```

**Replace with:**
```toml
[mcp_servers.ruflo]
command = "ruflo"
args = ["mcp", "start"]
```

All other entries (`notion`, model settings) are untouched.

---

#### Cursor — `~/.cursor/mcp.json`

**Current state:** has one Databricks entry

**Add ruflo alongside it:**
```json
{
  "mcpServers": {
    "dataplane-mcp-billing-usage": {
      "url": "https://dbc-82df9441-5364.cloud.databricks.com/api/2.0/mcp/genie/01f0530fd7291aa2bd7429c8fac9d2a8",
      "headers": { "Authorization": "Bearer <token>" }
    },
    "ruflo": {
      "command": "ruflo",
      "args": ["mcp", "start"]
    }
  }
}
```

---

#### Claude Code — terminal

Claude Code manages MCP via CLI. It also supports `pre/post-tool-use` hooks
— the only tool where ruflo fires automatically without explicit prompting.

**Step 4a — Register ruflo MCP**

```bash
# Check if stale claude-flow entry exists
claude mcp list

# Remove stale claude-flow entry if present (was in ~/.mcp.json global scope)
# Note: claude mcp remove only targets project/user scope — check ~/.mcp.json too
cat ~/.mcp.json  # if it has claude-flow, replace with ruflo manually

# Add ruflo to Claude Code
claude mcp add ruflo -- ruflo mcp start

# Verify — should show ruflo ✓ Connected, no claude-flow
claude mcp list
```

**Step 4b — Generate hook handler files**

Ruflo's hooks require helper files in `.claude/helpers/`. Run from your
project root:

```bash
# Generate hook handlers and settings.json with 7 hook types
ruflo init hooks --force

# Generate the full set of helper files (hook-handler.cjs, auto-memory-hook.mjs, statusline.cjs etc)
ruflo init --only-claude --force

# Verify helper files exist
ls .claude/helpers/hook-handler.cjs .claude/helpers/auto-memory-hook.mjs .claude/helpers/statusline.cjs
```

**Step 4c — Clean up settings.json**

`ruflo init` generates `.claude/settings.json` with stale claude-flow
branding and hardcoded model preferences. Edit to clean up:

- Remove `modelPreferences` block (let Claude Code use your authenticated model)
- Update `permissions.allow` — replace `npx @claude-flow*` / `npx claude-flow*`
  with `Bash(ruflo *)` and `Bash(npx ruflo*)`
- Update `attribution` — replace claude-flow branding with ruflo

Final clean `settings.json` structure:

```json
{
  "hooks": { ... 7 hook types referencing .claude/helpers/ ... },
  "statusLine": { "type": "command", "command": "node .claude/helpers/statusline.cjs" },
  "permissions": {
    "allow": ["Bash(ruflo *)", "Bash(npx ruflo*)", "Bash(node .claude/*)"],
    "deny": ["Read(./.env)", "Read(./.env.*)"]
  },
  "attribution": {
    "commit": "Co-Authored-By: ruflo <ruv@ruv.net>",
    "pr": "Generated with [ruflo](https://github.com/ruvnet/claude-flow)"
  },
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "CLAUDE_FLOW_V3_ENABLED": "true",
    "CLAUDE_FLOW_HOOKS_ENABLED": "true"
  },
  "claudeFlow": { ... swarm, memory, daemon, learning config ... }
}
```

**What the 7 hooks do once active:**

| Hook | Trigger | What ruflo does |
|---|---|---|
| `PreToolUse` (Bash) | Before any bash command | Risk assessment via `hook-handler.cjs pre-bash` |
| `PostToolUse` (Write/Edit) | After any file edit | Pattern stored via `hook-handler.cjs post-edit` |
| `UserPromptSubmit` | Every prompt submitted | Model routing via `hook-handler.cjs route` |
| `SessionStart` | Claude Code session opens | Memory imported + session restored |
| `SessionEnd` | Claude Code session closes | Memory synced to ruflo store |
| `Stop` | Agent stops | Memory synced via `auto-memory-hook.mjs sync` |
| `PreCompact` | Before context compaction | Session state saved |

---

### Step 5 — Restart all tools and verify

After restarting each tool, confirm ruflo MCP tools are visible using the
tool-specific commands below.

#### OpenCode

OpenCode does not have a CLI tool-list command. Verify by opening a session
and checking the tool picker, or inspect the MCP server status:

```bash
# Confirm the MCP server process starts cleanly
ruflo mcp start &
ruflo mcp status
ruflo mcp health
```

You should see `memory_search`, `swarm_init`, `agent_spawn` in the OpenCode
tool picker (accessible via `ctrl+p → tools`).

---

#### Codex

```bash
# List all registered MCP servers — ruflo should appear, claude-flow should not
codex mcp list

# If codex CLI is not available, check the config directly
cat ~/.codex/config.toml | grep mcp_servers -A 3
```

Expected output:
```
[mcp_servers.ruflo]
command = "ruflo"
args = ["mcp", "start"]
```

---

#### Cursor

Cursor does not have a CLI verification command. After restarting:
1. Open Cursor → Settings → MCP
2. Confirm `ruflo` appears in the MCP server list with status `connected`
3. Use native Cursor Agent mode — ruflo tools are available without any extension

Verify the config is correct:
```bash
cat ~/.cursor/mcp.json | python3 -m json.tool | grep -A 3 "ruflo"
```

Expected output:
```json
"ruflo": {
  "command": "ruflo",
  "args": ["mcp", "start"]
```

---

#### Claude Code

```bash
# List all registered MCP servers
claude mcp list

# Expected output:
# ruflo: ruflo mcp start - ✓ Connected
# claude-flow should NOT appear

# Verify hook handler files exist
ls .claude/helpers/hook-handler.cjs .claude/helpers/auto-memory-hook.mjs

# Test hooks fire without errors
node .claude/helpers/hook-handler.cjs session-restore
# Expected: "Session started: session-xxxx"

node .claude/helpers/auto-memory-hook.mjs import
# Expected: "[AutoMemory] Importing..." — no fatal errors
```

---

#### Universal check — confirm memory store is reachable

Run this from any terminal regardless of which tool you are in:

```bash
# Check MCP server health directly
ruflo mcp health

# Confirm memory store is initialised
ls -la ~/.claude/memory/

# Run a test memory store + search round-trip
ruflo memory store --key "test-setup" --value "ruflo mcp working" --namespace test
ruflo memory search --query "ruflo mcp working"
ruflo memory delete --key "test-setup" --namespace test
```

A successful search returning the stored value confirms the full stack
(MCP server → memory DB → HNSW index) is working end-to-end.

---

## 4. SDLC Phase Usage

RuFlo's value is **uneven across the SDLC**. Be realistic about where it helps.

### Planning & Architecture

**Useful — high value**

- `memory_search` before starting any task — retrieves past architectural
  decisions, patterns, and conventions so you don't re-litigate them
- `hooks_route` — tells you upfront if a task needs Opus-level reasoning or
  can be handled cheaper before you start burning tokens
- Spawn a `sparc-coord` + `architecture` swarm to break down a feature into
  ADRs before writing any code

```bash
# Check what's known about auth patterns before designing new auth flow
ruflo memory search --query "authentication patterns horizon-grid"

# Get routing recommendation before starting
ruflo hooks route --task "Design multi-tenant RBAC system"
```

---

### Implementation

**Strongest phase — highest ROI**

- **WASM Agent Booster**: simple transforms (var→const, add-types, async-await,
  remove-console) skip the LLM entirely — $0, <1ms
- **Parallel agents**: `agent_spawn coder` + `agent_spawn reviewer`
  simultaneously — actual parallel work
- **Pattern reuse**: `memory_search` before editing a file surfaces how similar
  problems were solved before
- **Store after success**: `memory_store` after completing a pattern — builds up
  project-specific knowledge over time

```bash
# Before implementing a feature — check what's been done before
ruflo memory search --query "NestJS guard implementation"

# Get model routing hint for the task
ruflo hooks model-route --task "Add rate limiting middleware"
# Returns: [TASK_MODEL_RECOMMENDATION] Use model="sonnet"

# After successful implementation — store the pattern
ruflo memory store \
  --key "rate-limiting-nestjs" \
  --value "ThrottlerGuard with custom storage adapter" \
  --namespace patterns
```

---

### Code Review

**Useful**

- Spawn `reviewer` + `security-auditor` agents in parallel
- `hooks_pre-edit` surfaces relevant past patterns before a file is touched
- `memory_search` for known issues in the area being reviewed

```bash
ruflo agent spawn -t reviewer --name pr-reviewer
ruflo agent spawn -t security-auditor --name sec-check
```

---

### Testing

**Moderately useful**

- `hooks_coverage-gaps` — identifies untested paths, prioritises by risk,
  assigns agents to fill them. Genuinely useful on a large codebase.
- `hooks_coverage-route` — routes test generation to the right agent
- `agent_spawn tester` for parallel test generation

```bash
ruflo hooks coverage-gaps --path backend/src/modules/auth
```

---

### Security Review

**Useful**

- `agent_spawn security-architect` + `agent_spawn security-auditor` in a swarm
- Built-in CVE scanning via `ruflo performance metrics` (shows CVE count)
- `hooks_pre-command` assesses risk before running commands

---

### Deployment & CI/CD

**Weak — limited value**

RuFlo has GitHub agents (`pr-manager`, `release-manager`) but these generate
config files rather than integrating natively with pipelines. No native support
for ArgoCD, GitHub Actions, or infrastructure tooling.

**Recommendation**: do not use ruflo for deployment. Use your existing CI/CD.

---

### Production Monitoring

**Not applicable**

RuFlo has no hooks into runtime telemetry, logs, or alerts. Completely outside
its scope.

---

### Phase Summary

| SDLC Phase | RuFlo Value | Key Tools |
|---|---|---|
| Planning | High | `memory_search`, `hooks_route`, `sparc-coord` swarm |
| Implementation | Highest | WASM booster, `agent_spawn`, `memory_store/search` |
| Code Review | Medium | `agent_spawn reviewer/security-auditor` |
| Testing | Medium | `hooks_coverage-gaps`, `agent_spawn tester` |
| Security | Medium | `agent_spawn security-auditor`, CVE scan |
| Deployment | Low | Not recommended — use existing CI/CD |
| Prod Monitoring | None | Out of scope |

---

## 5. Observations & Things to Keep in Mind

### The "313 tools" claim is inflated

Marketing says 313 tools. Actual enabled count when you run `ruflo mcp tools`
is **215**, across 20 categories. The delta includes disabled and deprecated
entries. Still a large number — see context overhead note below.

### Tools are not "triggered" automatically

MCP tools do not run in the background. What happens at session start:

1. Each tool's **name + description** (~20 tokens) is injected into the system
   prompt so the LLM knows the tools exist
2. A tool only **executes** when the LLM decides to call it, or you call it
   explicitly
3. At 215 tools × ~20 tokens = **~4,300 tokens of overhead per session**
   with all tools enabled
4. Trimmed to 34 essential tools = **~680 tokens overhead** — much more
   reasonable

### WASM auto-routing only works in Claude Code

The `[AGENT_BOOSTER_AVAILABLE]` and `[TASK_MODEL_RECOMMENDATION]` signals
appear in hook output, but the auto-routing hooks that fire them are
Claude Code-specific (`pre-tool-use`, `post-tool-use`, `UserPromptSubmit`).

These hooks are now **active** in horizon-grid via `.claude/settings.json`.
Every prompt submitted in Claude Code goes through `hook-handler.cjs route`
which calls ruflo's model router and injects the recommendation into context.

In OpenCode, Codex, and Cursor you must call `hooks_route` or
`hooks_model-route` explicitly via MCP to get routing recommendations.

### Codex stale entry must be removed, not renamed

`~/.codex/config.toml` currently has `[mcp_servers.claude-flow]`. If you add
`[mcp_servers.ruflo]` without removing the old entry, Codex spawns two MCP
processes pointing at the same memory DB — causes write contention and
confusing duplicate tool listings.

### Memory is empty on day one

All pattern learning metrics start at zero. The memory store only becomes
valuable after several real sessions have run through the MCP. The "32.3%
fewer tokens" figure from the dashboard is only meaningful once the
ReasoningBank has accumulated real usage data. Expect 1–2 weeks of regular
use before measurable token savings appear.

### The memory path is Claude Code-branded

RuFlo stores everything at `~/.claude/memory/`. This is a cosmetic quirk from
its claude-flow origins — there is no functional issue. All four tools write
to and read from the same path.

### Pin the version if stability matters

RuFlo's default branch is `dev` and it moves fast (v3.5.2 as of 2026-04-20,
6000+ commits). If you want predictable behaviour, pin in each tool config:

```bash
# Instead of ruflo@latest
npm install -g ruflo@3.5.2
```

And use `ruflo` (no version suffix) in all config files — the global install
controls the version.

### No cross-tool metrics breakdown

The metrics dashboard (`ruflo hooks metrics --v3-dashboard`) is one aggregate
DB. There is no breakdown of "tokens saved in OpenCode vs Cursor" — it's all
one pool. You cannot attribute savings to a specific tool.

---

## 6. Metrics & Observability

All metrics commands are available immediately after setup. Numbers populate
after real sessions run through the MCP.

| Command | What it shows |
|---|---|
| `ruflo hooks metrics --v3-dashboard` | 24h summary: patterns, routing accuracy, token reduction, command success rate |
| `ruflo hooks model-stats` | Per-model distribution (WASM/Haiku/Sonnet/Opus), cost savings vs all-Opus |
| `ruflo hooks intelligence --status` | SONA learning, MoE routing accuracy, HNSW index size |
| `ruflo performance metrics` | Heap/RSS memory, event loop latency, embedding cache hits |
| `ruflo hooks statusline` | Compact one-liner for shell prompt / tmux statusline |

**Metrics caveats:**

| Gap | Detail |
|---|---|
| Zeroed until first use | All counters start at 0 |
| No cross-tool breakdown | One aggregate DB, no per-tool split |
| No web dashboard | CLI only, no browser UI |
| 24h window fixed | No configurable time range |
| Token savings are pattern-based | Only meaningful once memory store has real data |

---

## 7. Day-to-Day Workflow Per Tool

> The core question: once ruflo is installed, how do you actually use it?
> There is no ruflo menu, no skill picker, no separate terminal to open.
> Ruflo's tools are available to the LLM like any other tool — the difference
> is in how each tool exposes and invokes them.

---

### The Three Usage Patterns (apply to all tools)

**Pattern 1 — Transparent / automatic**
The LLM calls ruflo tools on its own (memory_search before acting,
memory_store after success). Requires a system prompt or AGENTS.md that
instructs the agent to use memory tools. Without that instruction the LLM
will mostly ignore ruflo.

**Pattern 2 — Prompt-driven (works from day one)**
You explicitly mention ruflo tools in your message:
```
Before you start, search memory for how we handle auth in this project,
then implement rate limiting on the login endpoint.
```
Less automatic, but reliable immediately without any extra config.

**Pattern 3 — Custom agent (best long-term)**
Create a ruflo-aware agent whose system prompt always searches memory first
and stores patterns after. Set it as default. Ruflo runs silently on every
task. Config varies per tool — see below.

---

### OpenCode

**How ruflo surfaces**: As MCP tools available to the Build/Plan agents and
any custom agent. No separate menu — tools appear in the tool picker
(`ctrl+p → tools`).

**Entry points:**

| Action | How |
|---|---|
| Switch agent | `Tab` key to cycle, or `@agent-name` in prompt |
| Invoke ruflo tool explicitly | Mention it in your prompt: "search memory for X" |
| See available MCP tools | `ctrl+p` → tools list |
| Create ruflo-aware agent | Write `~/.config/opencode/agents/ruflo-build.md` |

**The missing piece**: OpenCode has no auto-hook system like Claude Code's
`pre-tool-use`. Ruflo does not fire automatically before every file edit.
You need either a custom agent system prompt, or explicit prompts.

**Recommended custom agent** — create this file:

`~/.config/opencode/agents/ruflo-build.md`
```markdown
---
description: Default build agent with ruflo memory and routing
mode: primary
model: anthropic/claude-sonnet-4-20250514
---
You have access to ruflo MCP tools. Follow this workflow on every task:

1. BEFORE starting: call memory_search with a query relevant to the task
2. Call hooks_model-route to get the optimal model tier recommendation
3. Do the work using the routing recommendation
4. AFTER success: call memory_store to save the pattern for future sessions

Never skip steps 1 and 4. They are what make this agent improve over time.
```

Invoke it with `@ruflo-build` or set it as your default primary agent in
`~/.config/opencode/opencode.jsonc`:
```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "default": "ruflo-build"
  },
  "mcp": {
    "ruflo": {
      "type": "local",
      "command": "ruflo",
      "args": ["mcp", "start"],
      "enabled": true
    }
  }
}
```

---

### Claude Code

**How ruflo surfaces**: As MCP tools. Claude Code also supports
`pre-tool-use` and `post-tool-use` hooks — this is the only tool where
ruflo can auto-fire WASM routing without explicit prompting.

**Entry points:**

| Action | How |
|---|---|
| Use ruflo tools | LLM calls them automatically (with hooks) or you mention them |
| WASM auto-routing | Fires via `pre-tool-use` hook — works natively here only |
| Check MCP tools | `claude mcp list` |
| Verify ruflo active | Start a session, ruflo tools appear in tool list |

**Workflow**: Claude Code is the only tool where ruflo works fully as
designed. The `pre-tool-use` hook intercepts every tool call, checks
`[AGENT_BOOSTER_AVAILABLE]`, and routes to WASM/Haiku/Opus automatically.
No custom agent or explicit prompting required.

**Recommended CLAUDE.md addition** for the project root:
```markdown
## RuFlo Memory Workflow
- ALWAYS call memory_search before starting any non-trivial task
- ALWAYS call memory_store after a successful pattern is established
- Use hooks_model-route to determine model tier before spawning agents
- Use swarm_init for tasks touching 3+ files
```

---

### Codex

**How ruflo surfaces**: As MCP server tools declared in `~/.codex/config.toml`.
Codex is headless and often used for automated/parallel tasks — ruflo's
memory tools are particularly useful here for maintaining context across
automated runs.

**Entry points:**

| Action | How |
|---|---|
| Use ruflo tools | Mention in task prompt or add to task template |
| Check MCP registered | `codex mcp list` |
| Verify config | `cat ~/.codex/config.toml \| grep mcp_servers -A 3` |

**Workflow**: Because Codex runs tasks autonomously, the most useful pattern
is to prepend every task with a memory search instruction via your
`~/.codex/AGENTS.md`:

```markdown
## Memory-First Workflow
Before starting any task:
1. Call memory_search with a query relevant to what you are about to do
2. Review results and incorporate known patterns
3. After completing: call memory_store to save what worked

This applies to every task without exception.
```

**Important**: Codex currently has a stale `[mcp_servers.claude-flow]` entry.
Remove it before adding ruflo — having both active spawns two MCP processes
and causes write contention on the shared memory DB.

---

### Cursor

**How ruflo surfaces**: As MCP tools available to Cursor's AI and to the
Roo-Cline extension (which you have installed). Roo-Cline has the richest
MCP integration of all Cursor extensions — it shows available tools, lets
you approve/deny calls, and supports `@` mentions of MCP servers.

**Entry points:**

| Action | How |
|---|---|
| Use ruflo in Cursor AI chat | Mention in prompt: "search memory for X" |
| Use ruflo in Roo-Cline | `@ruflo` in chat to reference the MCP server |
| See available tools | Roo-Cline settings → MCP → ruflo → tool list |
| Approve/deny tool calls | Roo-Cline shows a confirmation dialog per MCP call |

**Workflow**: Cursor is best used for implementation work where you want
visual feedback. The recommended pattern:

1. Open Roo-Cline, type your task
2. Prefix with: "First search ruflo memory for relevant patterns, then..."
3. Roo-Cline will show you each ruflo tool call before executing — approve them
4. After completion, ask it to store the pattern: "Store this approach in ruflo memory"

**Note on Roo-Cline vs native Cursor AI**: Native Cursor AI (the sidebar chat)
also has access to ruflo MCP tools but does not show tool call confirmations.
Roo-Cline gives more visibility and control over what ruflo is doing.

---

### Summary: workflow comparison across tools

| Tool | Ruflo auto-fires? | Entry point | Best for |
|---|---|---|---|
| **Claude Code** | Yes (via hooks) | Automatic + CLAUDE.md | Full autonomous workflow |
| **OpenCode** | No — needs custom agent | `@ruflo-build` agent or explicit prompts | Interactive development |
| **Codex** | No — needs AGENTS.md | `~/.codex/AGENTS.md` instruction | Automated / headless tasks |
| **Cursor** | No — needs explicit prompt | Roo-Cline `@ruflo` or prompt mention | Visual/interactive implementation |

---

## 8. Reference

| Item | Value |
|---|---|
| RuFlo repo | https://github.com/ruvnet/claude-flow |
| npm package | https://www.npmjs.com/package/ruflo |
| Current version | v3.5.2 (2026-04-20) |
| Old package name | `claude-flow` / `@claude-flow/cli` (same binary, rebranded) |
| Memory store path | `~/.claude/memory/` |
| OpenCode MCP docs | https://opencode.ai/docs/mcp-servers/ |
| Codex MCP config | `~/.codex/config.toml` → `[mcp_servers.<name>]` |
| Cursor MCP config | `~/.cursor/mcp.json` → `mcpServers` |
| Claude Code MCP | `claude mcp add <name> -- <command>` |
