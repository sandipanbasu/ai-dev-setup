# AI-Assisted Development Lifecycle with RuFlo

> How to use RuFlo v3.5 as the memory and orchestration backbone across
> OpenCode, Claude Code, Codex, and Cursor — with specific workflows for
> each tool at every phase of development.
>
> Last updated: 2026-04-20 | RuFlo: v3.5.2 | Project: horizon-grid

---

## The Core Idea

Every AI coding session starts cold. The agent has no memory of yesterday's
decisions, re-reads entire files to get context, and burns expensive tokens
on things it already figured out last week.

RuFlo fixes this by acting as a **shared memory and routing layer** across
all four tools. One SQLite + HNSW database at `~/.claude/memory.db`,
accessible by every tool, persisting across all sessions forever.

The value **compounds over time**:

```
Day 1:    7 patterns  — seeded baseline (architecture, modules, auth, DB, jobs)
Week 2:  ~20 patterns — first 3 features completed
Week 4:  ~50 patterns — real project knowledge accumulating
Month 3: ~200 patterns — agent knows the codebase better than a new developer
```

---

## How RuFlo Surfaces in Each Tool

| Tool | MCP config | Auto-fires? | Entry point |
|---|---|---|---|
| **OpenCode** | `~/.config/opencode/opencode.jsonc` | No — needs `@ruflo-build` agent | `@ruflo-build` in chat |
| **Claude Code** | `~/.mcp.json` + `claude mcp add` | Yes — `pre/post-tool-use` hooks | Fully automatic |
| **Codex** | `~/.codex/config.toml` (`[mcp_servers.ruflo]`) | No — needs `AGENTS.md` instruction | `AGENTS.md` + plain prompts + `ruflo` CLI |
| **Cursor** | `~/.cursor/mcp.json` | No — needs explicit prompt | Native Cursor AI chat (Agent mode) |

---

## The Three Daily Habits (apply to all tools)

**1. Start with memory**
Before any task, search memory for relevant context. Automatic in Claude Code
and `@ruflo-build`. Manual prompt needed in Codex and Cursor.

**2. End with a store**
One sentence at the end of every session:
```
store what we built today in ruflo memory
```

**3. Store decisions explicitly**
Any non-obvious design decision:
```
store the decision to use X because Y in ruflo memory
```

---

## Workspace Setup (run once when onboarding a new workspace)

Before using ruflo with any workspace, run through this setup checklist.
It establishes the branch gates, plan folder, and memory seeds that every
agent across all tools will rely on.

### 1. Identify protected branches

```bash
# Check what the main and integration branches are
git branch -a | head -20
cat .git/config | grep -E "branch|merge"
```

Store the answer in ruflo memory:

```bash
ruflo memory store \
  --key "<workspace>-git-config" \
  --value "main branch = <main>, integration branch = <develop>, remote = origin (<url>). Protected: <main>, <develop>. Branch from <develop>. Naming: feature/<name>, fix/<name>, chore/<name>." \
  --namespace project
```

**horizon-grid values:**
- Protected branches: `main`, `develop`
- Branch from: `develop`
- Remote: GitLab `gitlab.com/skyspecs/data-engg/horizon-grid`

### 2. Create the plans folder

```bash
mkdir -p ~/ruflo-plans/<workspace-name>
```

**horizon-grid:** `~/ruflo-plans/horizon-grid/`

Store in ruflo memory:

```bash
ruflo memory store \
  --key "feature-plan-location" \
  --value "Feature plans stored at ~/ruflo-plans/<workspace>/. Never inside repo. One file per feature: <feature-name>.md" \
  --namespace project
```

### 3. Seed project memory

Run from inside the workspace:

```bash
ruflo memory store --key "<workspace>-architecture" --value "..." --namespace project
ruflo memory store --key "<workspace>-backend-modules" --value "..." --namespace project
# etc — see Current Memory Seeds section below
```

### 4. Add branch gates to Claude Code settings

Add to `.claude/settings.json` `permissions.deny`:

```json
"Bash(git push * main*)",
"Bash(git push * develop*)",
"Bash(git push --force*)",
"Bash(git push -f *)",
"Bash(git checkout main*)",
"Bash(git checkout develop*)",
"Bash(git merge main*)",
"Bash(git merge develop*)",
"Bash(git reset --hard*)",
"Bash(git rebase main*)",
"Bash(git rebase develop*)"
```

This makes Claude Code **refuse** to run any of these commands. The hook
fires before execution — the agent cannot override it.

### 5. Branch gate is enforced in the ruflo-build agent

The `@ruflo-build` agent in OpenCode checks the current branch on every
session start. If you are on `main` or `develop` it stops and asks you
to create a feature branch before proceeding. It will not write a single
file until you are on a feature branch.

---

## Feature Lifecycle — Branch, Plan, Approve, Implement, Test, PR

Every feature follows this structure regardless of which tool you use.

### Plan file convention

Feature plans are stored **outside the repo** in a central plans directory:

```
~/ruflo-plans/
└── horizon-grid/
    ├── flight-approval-workflow.md
    ├── digital-twin-sync.md
    └── ...
```

- Never store plans inside the repo
- One markdown file per feature
- Naming: `<feature-name>.md`
- This location is stored in ruflo memory (`feature-plan-location`) so any
  agent across any tool knows where to write and read plans

---

### Step 1 — Create branch

Ruflo has no auto-branch tool. The agent runs git via Bash. Ask it:

```
@ruflo-build create a branch called feature/flight-approval-workflow
```

Agent runs:
```bash
git checkout -b feature/flight-approval-workflow
```

`hooks_pre-command` assesses risk before executing. Branch is created.

---

### Step 2 — Write the plan

```
@ruflo-build search memory for relevant patterns and write a feature plan
for flight-approval-workflow to ~/ruflo-plans/horizon-grid/flight-approval-workflow.md
```

Agent:
1. `memory_search` — pulls existing patterns (modules, auth, schema)
2. `hooks_model-route` — recommends Opus for design tasks
3. Writes structured plan to `~/ruflo-plans/horizon-grid/flight-approval-workflow.md`
4. `memory_store` — saves the plan reference

Plan file structure the agent writes:

```markdown
# Feature: Flight Approval Workflow

## Summary
## Affected modules
## Schema changes
## API surface
## Task breakdown
## Open questions
## Risks
```

---

### Step 3 — Review & approve (human step)

**Ruflo has no built-in approval gate.** This is intentional — you are
the architect. Open the plan file, read it, refine it, then explicitly
tell the agent to proceed:

```
I've reviewed ~/ruflo-plans/horizon-grid/flight-approval-workflow.md
and approved it. Proceed with implementation.
```

The agent will not start implementing until you say this. This is the
gate between planning and building.

---

### Step 4 — Implementation

```
@ruflo-build implement the plan at ~/ruflo-plans/horizon-grid/flight-approval-workflow.md
```

Agent reads the plan, searches memory for patterns, implements with correct
model routing per task complexity.

---

### Step 5 — Testing

```
@ruflo-build write tests for everything implemented in this branch
```

---

### Step 6 — Pre-PR diff review

Before raising a PR, ruflo analyses what changed:

```
@ruflo-build analyse the diff on this branch and assess risk
```

Agent calls:
- `analyze_diff` — risk level, change classification
- `analyze_diff-reviewers` — who should review based on changed files
- `analyze_diff-stats` — lines changed, files affected

---

### Step 7 — Raise PR

```
@ruflo-build create a PR for this branch
```

Agent calls `gitlab_pr_manage` — creates the PR on GitLab with title, body
populated from the plan file, and correct base branch.

---

### Step 8 — End of session store

```
@ruflo-build store what we built today in ruflo memory
```

---

---

### On Session Start

**OpenCode**
Ruflo MCP spawns automatically when OpenCode loads. No action needed.
Verify in logs: `~/.local/share/opencode/log/<latest>.log` — look for
`key=ruflo toolCount=215 create() successfully created client`.

**Claude Code**
Ruflo MCP spawns automatically. Hooks fire on every tool use.
Verify: `claude mcp list` → `ruflo: ruflo mcp start - ✓ Connected`

**Codex**
Ruflo MCP is registered in `~/.codex/config.toml` under `[mcp_servers.ruflo]`.
Codex also has a dedicated adapter bundled inside ruflo: `@claude-flow/codex`.
Verify: `codex mcp list` → `ruflo  ruflo  mcp start  enabled`

**Cursor**
Ruflo MCP loads on Cursor start via `~/.cursor/mcp.json`. The native
Cursor AI (Agent mode in the chat sidebar) reads this config directly —
no extension required.
Verify: Cursor → Settings → MCP → ruflo should show as connected.
In Agent mode, ruflo tools are available alongside built-in tools.

---

### Phase 1: Planning

> Goal: understand the feature scope, surface relevant existing patterns,
> decide on approach — without reading files.

---

**OpenCode**

Switch to `@ruflo-build` agent (Tab or type `@ruflo-build`):

```
@ruflo-build I need to add a flight mission approval workflow
with multi-stage sign-off by account managers
```

Agent automatically:
1. Calls `memory_search("flight planning workflow patterns")` — surfaces
   `horizon-grid-backend-modules`, `horizon-grid-auth-pattern`
2. Calls `hooks_model-route("design multi-stage approval workflow")` — recommends Opus
3. Produces a feature breakdown grounded in actual project context

End of planning — store the decision:
```
store this planning decision in ruflo memory
```

---

**Claude Code**

Type your task normally — hooks handle memory automatically:

```
I need to add a flight mission approval workflow with multi-stage sign-off
```

Pre-tool-use hooks fire before any file is read:
- `memory_search` runs automatically
- `[TASK_MODEL_RECOMMENDATION]` signal appears in context
- Agent proceeds with memory context before touching any file

No extra prompting needed. This is the only tool where ruflo is truly
hands-free.

Store decision at end:
```
store the planning decision in ruflo memory
```

---

**Codex**

Codex has two real integration points with ruflo:
1. **MCP tools** — `memory_search`, `memory_store` etc called automatically
   per the `~/.codex/AGENTS.md` memory-first instruction
2. **CLI** — `ruflo sparc run <mode> "task"` for structured SPARC workflows

For planning, prompt directly — Codex calls `memory_search` per AGENTS.md:

```bash
codex "plan the flight mission approval workflow feature"
```

For a more structured breakdown using SPARC methodology via CLI:

```bash
ruflo sparc run spec-pseudocode "flight mission approval workflow with multi-stage sign-off"
```

Pipe output directly to the plans folder:

```bash
codex "plan the approval workflow using patterns from ruflo memory.
Output a structured task breakdown." > ~/ruflo-plans/horizon-grid/flight-approval-workflow.md
```

---

**Cursor**

Use Cursor's native Agent mode (not Roo-Cline). Switch to Agent mode in
the chat sidebar, then prefix your task with an explicit memory search:

```
First search ruflo memory for flight planning and auth patterns in this
project, then help me plan a multi-stage approval workflow feature.
```

Cursor Agent calls `memory_search` via the MCP connection in
`~/.cursor/mcp.json`. Results appear inline before the plan is generated.

Store decision at end:
```
store this planning decision in ruflo memory
```

---

### Phase 2: Architecture & Design

> Goal: identify affected modules, schema changes, API surface — grounded
> in project conventions, not guesswork.

---

**OpenCode**

```
@ruflo-build break down the approval workflow into affected modules,
schema changes needed, and API endpoints
```

Agent calls:
- `memory_search("existing modules affected by approval")` → flight-planning, work-orders, notification
- `memory_search("prisma schema conventions")` → PostGIS, horizongrid schema, @@map conventions
- `memory_search("auth RBAC pattern")` → isAccountManager, UserRole junction

Produces an ADR or task list without reading any files.

Store the architecture decision:
```
store the approval workflow architecture decision in ruflo memory
```

---

**Claude Code**

Architecture work happens automatically. Just describe what you need:

```
Design the DB schema and module structure for the approval workflow feature.
Reference existing patterns in the project.
```

Pre-tool-use hook fires `memory_search` before every file read or write.
Agent pulls conventions from memory first, only reads files when memory
is insufficient.

---

**Codex**

Best for generating boilerplate from architecture decisions stored in memory.
Prompt directly — Codex reads memory first per AGENTS.md:

```bash
codex "using the approval workflow architecture stored in ruflo memory,
generate the NestJS module skeleton for ApprovalModule"
```

For structured ADR output, use the ruflo CLI directly:

```bash
ruflo sparc run architect "ApprovalModule for flight-planning: state machine, REST API, Prisma model"
```

Codex generates scaffolding consistent with the project's existing module
conventions pulled from ruflo memory.

---

**Cursor**

Use Cursor Agent mode for architecture exploration — the chat stays open
alongside the file tree and diff views. Ask it to pull context from memory
before opening files:

```
Search ruflo memory for the approval workflow architecture decision,
then show me which files in the flight-planning module need to change
```

Cursor Agent calls `memory_search`, surfaces the stored architecture
decision, then navigates to the relevant files in the editor — combining
memory context with visual file browsing.

---

### Phase 3: Implementation

> Goal: write code consistent with project patterns, minimal token waste,
> correct model for the task complexity.

---

**OpenCode**

Primary implementation tool. `@ruflo-build` handles routing automatically:

```
@ruflo-build implement the ApprovalService in the flight-planning module
```

On every file touch, agent:
1. `memory_search("NestJS service pattern")` → recalls module conventions
2. `hooks_model-route("implement ApprovalService")` → recommends Sonnet
3. Writes code, stores successful pattern:
   `memory_store("approval-service-pattern", "...")`

For parallel subtasks within a feature:
```
@ruflo-build implement these 3 things in parallel:
1. ApprovalService with state machine
2. ApprovalController with REST endpoints  
3. Prisma migration for ApprovalStage model
```

---

**Claude Code**

Best for complex, multi-file implementation where hooks add the most value.
Every file edit is preceded by a memory search, every successful edit
triggers a pattern store.

```
Implement the full approval workflow: service, controller, Prisma model,
and wire it into the flight-planning module.
```

WASM Agent Booster fires automatically for simple transforms (renaming,
type additions) — these skip the LLM entirely, $0 cost.

---

**Codex**

The strongest tool for parallel, headless implementation. Ruflo's
`@claude-flow/codex` adapter supports a dual-mode model where Claude Code
handles interactive reasoning and Codex handles parallel bulk work.

Simple prompt — Codex calls memory first per AGENTS.md:

```bash
codex "implement ApprovalService in flight-planning module using NestJS patterns from ruflo memory"
```

For structured SPARC-guided implementation, use the ruflo CLI:

```bash
ruflo sparc run coder "ApprovalService with state machine in flight-planning module"
```

Parallel tasks — each worker reads the same shared ruflo memory independently:

```bash
codex "implement ApprovalService using ruflo memory patterns" &
codex "implement ApprovalController using ruflo memory patterns" &
codex "generate Prisma migration using DB conventions from ruflo memory" &
wait
```

Dual-mode task routing (from ruflo's codex adapter):

| Task complexity | Platform | Reason |
|---|---|---|
| Simple (1-2 files) | Codex headless | Fast, parallel |
| Medium (3-5 files) | OpenCode/Claude Code | Needs context |
| Complex (architecture) | Claude Code | Reasoning required |
| Bulk (many files) | Codex workers | Parallelise |

---

**Cursor**

Best for implementation where you want visual feedback — see diffs before
accepting changes. Use Cursor Agent mode:

```
Search ruflo memory for NestJS service patterns, then help me implement
ApprovalService in backend/src/modules/flight-planning/
```

Cursor Agent calls `memory_search` via MCP, gets the pattern, then
generates code. Cursor's diff view shows proposed changes before applying.

After implementing each piece:
```
Store the ApprovalService implementation pattern in ruflo memory
```

---

### Phase 4: Testing

> Goal: write tests consistent with existing conventions, identify
> coverage gaps, no need to read existing tests for reference.

---

**OpenCode**

```
@ruflo-build write tests for ApprovalService
```

Agent calls:
- `memory_search("testing patterns jest")` → recalls: backend uses Jest,
  mocks via jest.mock(), test files in __tests__ or *.spec.ts
- `hooks_coverage-gaps` on the new module → identifies untested paths
- Writes tests matching existing conventions without reading other test files

---

**Claude Code**

```
Write comprehensive tests for the approval workflow implementation.
Follow the project's existing testing patterns.
```

Pre-tool-use hooks pull testing patterns from memory before reading any
test files. Coverage gaps are identified via `hooks_coverage-gaps` which
fires automatically for new files.

---

**Codex**

Strongest tool for parallel test generation. Prompt directly — Codex calls
memory first per AGENTS.md. Run all layers simultaneously:

```bash
codex "write unit tests for ApprovalService using jest patterns from ruflo memory" &
codex "write integration tests for ApprovalController using ruflo memory patterns" &
codex "write e2e tests for approval workflow using playwright patterns from ruflo memory" &
wait
```

For SPARC-structured test generation via ruflo CLI:

```bash
ruflo sparc run tester "ApprovalService unit tests — state machine transitions, RBAC guards"
```

Each parallel worker reads testing conventions from the shared ruflo memory
independently.

---

**Cursor**

Best for test writing when you want to see test and source file
side-by-side. Use Cursor Agent mode:

```
Search ruflo memory for testing patterns, then write tests for
ApprovalService. Open the source and test files side by side.
```

Cursor Agent calls `memory_search` for testing conventions, then generates
tests. Use Cursor's split editor to view source and test simultaneously.

---

### Phase 5: Code Review

> Goal: verify implementation matches project patterns, catch deviations
> before committing — without reading the entire codebase.

---

**OpenCode**

```
@ruflo-build review the approval workflow implementation against
project patterns
```

Agent checks from memory:
- RBAC applied correctly (vs `horizon-grid-auth-pattern`)
- Prisma model follows conventions (vs `horizon-grid-db-schema`)
- Notifications wired correctly (vs `horizon-grid-jobs-pattern`)
- Module structure consistent (vs `horizon-grid-backend-modules`)

Flags deviations as specific comments without reading every project file.

---

**Claude Code**

```
Review everything implemented for the approval workflow. Check against
existing project patterns and flag any deviations.
```

Pre-tool-use hooks pull all relevant patterns from memory before the
review starts. The agent reviews against memory context, only reads
files when it needs to verify something specific.

---

**Codex**

Best for automated pattern compliance checks — run headless before every PR:

```bash
codex "review the approval workflow implementation in backend/src/modules/flight-planning
against patterns stored in ruflo memory. Output a structured deviation report."
```

For SPARC-structured review output via ruflo CLI:

```bash
ruflo sparc run reviewer "ApprovalModule implementation review" > review-report.md
```

Output can be piped to a file or posted as a PR comment.

---

**Cursor**

Best for interactive review — see diffs while getting memory-informed
feedback. Use Cursor Agent mode:

```
Search ruflo memory for auth, schema, and notification patterns,
then review the approval workflow implementation for any deviations
```

Cursor Agent calls `memory_search`, loads the relevant patterns, then
highlights deviations in the editor with inline suggestions.

---

### Phase 6: End of Session — The Habit That Compounds

> This is the most important step across all tools. One prompt, every session.

---

**OpenCode / Claude Code / Cursor**

```
store what we built today in ruflo memory
```

Agent calls `memory_store` with a structured summary:

```
key: "flight-approval-implementation"
namespace: "patterns"
value: "Multi-stage approval via state machine in flight-planning module.
        States: PENDING → REVIEWED → APPROVED/REJECTED.
        NotificationModule for stage transitions.
        RBAC: isAccountManager required for final approval.
        Prisma: ApprovalStage model with @@schema('horizongrid').
        Tests: Jest unit + Playwright e2e."
```

---

**Codex**

Prompt directly — Codex will call `memory_store` per the AGENTS.md instruction:

```bash
codex "summarise what was implemented today for the approval workflow
and store it in ruflo memory"
```

Most reliable option — store directly via the ruflo CLI after any session:

```bash
ruflo memory store \
  --key "flight-approval-implementation" \
  --value "Multi-stage approval via state machine in flight-planning module.
States: PENDING → REVIEWED → APPROVED/REJECTED.
NotificationModule for transitions. isAccountManager required for final approval.
Prisma: ApprovalStage @@schema('horizongrid'). Tests: Jest + Playwright." \
  --namespace patterns
```

The CLI store works regardless of which tool was used during the session.

---

## Codex + RuFlo SPARC Modes

Ruflo's SPARC methodology is available via the CLI (`ruflo sparc run <mode>`)
or as MCP tools in Claude Code. The `$skill-name` syntax seen in some ruflo
README files is a Codex CLI convention for loading skill markdown files from
`.agents/skills/` — it is **not a callable command**, it is a hint to Codex
to read and follow that skill's instructions.

### How to invoke SPARC per tool

| Tool | How to invoke | Example |
|---|---|---|
| **Claude Code** | MCP tool call (automatic via hooks) | Agent calls `sparc_mode` automatically |
| **OpenCode** | Natural language prompt | `@ruflo-build design the module structure` |
| **Codex** | Plain prompt — reads AGENTS.md | `codex "implement X using ruflo memory patterns"` |
| **Terminal (any)** | ruflo CLI | `ruflo sparc run coder "task description"` |

### SPARC modes and their ruflo CLI commands

| Mode | CLI command | What it does |
|---|---|---|
| Specification | `ruflo sparc run spec-pseudocode "task"` | Capture context, write pseudocode |
| Architecture | `ruflo sparc run architect "task"` | System design, module structure |
| Implementation | `ruflo sparc run coder "task"` | Clean code generation |
| Testing | `ruflo sparc run tester "task"` | Comprehensive test writing |
| Review | `ruflo sparc run reviewer "task"` | Code review and quality check |
| Debug | `ruflo sparc run debugger "task"` | Bug investigation |
| Security | `ruflo sparc run security-review "task"` | Security audit |
| Docs | `ruflo sparc run documenter "task"` | Documentation generation |

List all available modes:
```bash
ruflo sparc modes
```

### Initialise skills for a project (run once per project, optional)

Only needed if you want Codex to load skill files from `.agents/skills/`
when you reference them in prompts:

```bash
# Minimal — core skills only
npx claude-flow@alpha init --codex --minimal

# Full — all 137+ skills
npx claude-flow@alpha init --codex --full

# Dual mode — sets up both Claude Code (CLAUDE.md) and Codex (AGENTS.md)
npx claude-flow@alpha init --dual
```

---

## Memory Namespaces

| Namespace | What goes here |
|---|---|
| `project` | Architecture, module list, build commands, stack overview |
| `patterns` | How specific problems are solved: auth, jobs, notifications, RBAC |

```bash
# Search by namespace
ruflo memory search --query "auth patterns" --namespace patterns
ruflo memory search --query "module structure" --namespace project

# List all stored entries
ruflo memory list --namespace project
ruflo memory list --namespace patterns
```

---

## Current Memory Seeds (horizon-grid, seeded 2026-04-20)

| Key | Namespace | Content |
|---|---|---|
| `horizon-grid-architecture` | project | Full stack overview |
| `horizon-grid-backend-modules` | project | All 22 NestJS modules |
| `horizon-grid-frontend-pages` | project | All React pages + stack |
| `horizon-grid-db-schema` | project | Prisma conventions + PostGIS |
| `horizon-grid-auth-pattern` | patterns | Auth0 + RBAC + throttling |
| `horizon-grid-build-commands` | project | All build/test/db scripts |
| `horizon-grid-jobs-pattern` | patterns | BullMQ + Redis architecture |

---

## Useful CLI Commands

```bash
# Verify ruflo is working
ruflo --version
ruflo memory stats

# Search memory
ruflo memory search --query "your question"
ruflo memory search --query "auth patterns" --namespace patterns

# Store a pattern manually
ruflo memory store --key "my-pattern" --value "description" --namespace patterns

# Check daily metrics (after sessions accumulate)
ruflo hooks metrics --v3-dashboard

# Get model routing recommendation
ruflo hooks model-route --task "describe your task"

# Check what tools are active
ruflo mcp tools | grep Enabled
```

---

## Related

- Installation and integration plan: `ruflo-mcp-integration-plan.md`
- RuFlo repo: https://github.com/ruvnet/claude-flow
- OpenCode agents docs: https://opencode.ai/docs/agents/
- OpenCode MCP docs: https://opencode.ai/docs/mcp-servers/
