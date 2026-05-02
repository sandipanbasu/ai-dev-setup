# Part 1: LLM Agents for Software Development — Fundamentals

> A foundational guide for developers on working effectively with AI coding agents.
> Understanding the mental models, concepts, and lifecycle of agent-assisted development.
>
> **Note:** This document is generic and tool-agnostic. For Parts 2-4, we use **RuFlo** as a concrete example of a Layer 5 orchestration system. You may choose RuFlo, another solution, or none at all — the concepts remain the same.
>
> Last updated: 2026-04-20

---

## The Stack: Where Everything Fits

Understanding the architecture prevents confusion about who does what. This stack represents the current landscape — each layer has multiple options.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  LAYER 5: ORCHESTRATION & WORKFLOW (Choose one, many, or none)              │
│  ─────────────────────────────────────────────────────────────               │
│                                                                             │
│  ┌──────────┐  ┌──────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  RuFlo   │  │ oh-my-opencode│  │ oh-my-codex │  │   mem0 / other      │  │
│  │          │  │              │  │             │  │   memory systems    │  │
│  │ • Memory │  │ • Swarm      │  │ • Skills    │  │   • Vector stores   │  │
│  │ • Routing│  │ • Routing    │  │ • Routing   │  │   • Session mgmt    │  │
│  │ • Safety │  │ • Safety     │  │ • Safety    │  │   • Retrieval       │  │
│  │ • Swarm  │  │ • Hooks      │  │ • Hooks     │  │                     │  │
│  └────┬─────┘  └──────┬───────┘  └──────┬──────┘  └──────────┬──────────┘  │
│       │               │                 │                    │             │
│       └───────────────┴─────────────────┴────────────────────┘             │
│                            Optional orchestration layer                     │
│                    Adds: memory, routing, safety, coordination              │
├─────────────────────────────────────────────────────────────────────────────┤
│  LAYER 4.5: CODE INTELLIGENCE & INDEXING (The Knowledge Base)               │
│  ─────────────────────────────────────────────────────────────               │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │  Sourcegraph │  │   Continue   │  │  GitHub      │  │  RuFlo       │    │
│  │  Cody        │  │   Index      │  │  Copilot     │  │  Pretrain    │    │
│  │              │  │              │  │  Chat        │  │              │    │
│  │ • Cross-repo │  │ • Local      │  │ • Codebase   │  │ • Pattern    │    │
│  │ • Symbol     │  │ • Embeddings │  │ • PR context │  │   extraction │    │
│  │   graph      │  │ • Open src   │  │ • VS Code    │  │ • Semantic   │    │
│  │ • Enterprise │  │              │  │   native     │  │   search     │    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
│         │                 │                 │                 │             │
│         └─────────────────┴─────────────────┴─────────────────┘             │
│  Ingests, chunks, embeds — makes code searchable by meaning                │
│  Bridge between raw code and agent understanding                           │
│  Agents query this layer to understand codebase before writing code        │
├─────────────────────────────────────────────────────────────────────────────┤
│  LAYER 4: CODING AGENT INTERFACES (Built-in orchestration emerging)         │
│  ─────────────────────────────────────────────────────────────────           │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  OpenCode   │  │ Claude Code │  │    Codex    │  │   Cursor    │        │
│  │  (VS Code)  │  │  (Terminal) │  │   (CLI)     │  │   (IDE)     │        │
│  │             │  │             │  │             │  │             │        │
│  │ Indexing:   │  │ Indexing:   │  │ Indexing:   │  │ Indexing:   │        │
│  │ None        │  │ Minimal     │  │ None        │  │ Strong      │        │
│  │             │  │ (CLAUDE.md) │  │             │  │ (native)    │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                 │               │
│         └────────────────┴────────────────┴─────────────────┘               │
│                            All can use Layer 4.5 or work standalone        │
├─────────────────────────────────────────────────────────────────────────────┤
│  LAYER 3: LLM PROXY / ROUTER (Your Org)                                     │
│  ─────────────────────────────────────                                       │
│  LiteLLM Proxy, Kong, custom gateway, or direct API                         │
│  • Unified API for multiple providers                                       │
│  • Model aliases, load balancing, rate limiting, cost tracking              │
│  • Routes to: Anthropic, OpenAI, Bedrock, Azure, etc.                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  LAYER 2: LLM PROVIDERS (External APIs)                                     │
│  ─────────────────────────────────────                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Anthropic  │  │   OpenAI    │  │    AWS      │  │   Azure     │        │
│  │  (Claude)   │  │  (GPT-4)    │  │ (Bedrock)   │  │  (OpenAI)   │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                 │               │
│         └────────────────┴────────────────┴─────────────────┘               │
│                    Provides raw LLM capabilities                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  LAYER 1: FOUNDATION MODELS                                                 │
│  ─────────────────────────                                                   │
│  Claude 4.6 Sonnet, Claude 4.7 Opus, GPT-5, Gemini, Llama, etc.             │
│  • Token prediction engines                                                  │
│  • Reasoning, code generation, analysis                                      │
│  • Stateless — no memory between API calls                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### What Each Layer Actually Does

| Layer | Component | What It Does | Who Controls It | Examples |
|---|---|---|---|---|
| **1** | Foundation Models | Raw text generation, reasoning | Provider | Claude, GPT-4, Gemini |
| **2** | LLM Providers | API endpoints, billing, rate limits | Provider | Anthropic API, OpenAI API |
| **3** | LLM Proxy/Router | Unified gateway, cost tracking | **Your org** | LiteLLM, Kong, custom |
| **4** | Coding Agents | IDE/editor integration, file access, git | Tool vendors | OpenCode, Claude Code, Codex, Cursor |
| **4.5** | Code Indexers | Ingest, chunk, embed, make code searchable | **You choose** | Sourcegraph Cody, Continue, GitHub Copilot Chat, RuFlo Indexer |
| **5** | Orchestration Layer | Cross-session memory, routing, safety, coordination | **You choose** | RuFlo, oh-my-opencode, oh-my-codex, mem0, none |


### Layer 4.5: Code Intelligence & Indexing (Augmentation Layer)

**The Reality:** Layer 4 tools have **mixed built-in indexing:**

| Tool | Built-in Indexing | Notes |
|---|---|---|
| **Cursor** | Strong | Native codebase indexing for Agent mode |
| **GitHub Copilot Chat** | Medium | Indexes via VS Code extension |
| **Claude Code** | Minimal | CLAUDE.md + file reading, not semantic search |
| **Codex** | None | Relies on commands and context7 |
| **OpenCode** | None | No built-in codebase indexing |

**When to Add Layer 4.5:**

Layer 4.5 provides **dedicated, tool-agnostic code intelligence** that:
- Works across all Layer 4 tools (one index, everywhere)
- More powerful than built-in (symbol graphs, cross-repo, PR context)
- Survives tool switches (index persists regardless of which IDE you use)

| Capability | Layer 4 Alone | With Layer 4.5 |
|---|---|---|
| Semantic search | Cursor only, others no | All tools |
| Symbol graph | Limited | Full cross-reference |
| Cross-repo intelligence | No | Yes |
| PR context awareness | No | Changed files + related code |
| Tool-agnostic | Each tool isolated | One index, all tools |

**Available tools:**
- **Sourcegraph Cody** - enterprise code intelligence, cross-repo, symbol graph
- **Continue Index** - local, open-source, embeddings-based
- **GitHub Copilot Chat** - codebase context, PR context, VS Code native
- **RuFlo Pretrain** - ruflo hooks_pretrain for pattern extraction and analysis

**Decision guide:**
- Use Layer 4 built-in indexing if you use Cursor primarily
- Add Layer 4.5 when you use multiple tools and need consistent intelligence across all of them
- Add Layer 4.5 when you need cross-repo search, symbol graphs, or PR context awareness

---

### Important Trends

**Trend 1: Layer 4 is absorbing Layer 5**

Coding agents are building orchestration capabilities directly in:
- OpenCode — native agents, memory, custom agent files
- Claude Code — `CLAUDE.md`, pre/post-tool hooks, session management
- Codex — commands, inline agents, AGENTS.md
- Cursor — Agent mode, native MCP support

This means Layer 5 is increasingly optional for single-tool workflows. However, it remains essential for **cross-tool consistency** — when you switch between OpenCode, Codex, Cursor, and Claude Code, built-in orchestration does not travel with you.

---

**Trend 2: Layer 4.5 is becoming standard**

Code indexing is rapidly moving from "nice to have" to "expected":
- Cursor ships with native codebase indexing — it is on by default
- GitHub Copilot Chat requires codebase indexing to be useful
- Continue.dev has made local indexing accessible and free
- Sourcegraph Cody is becoming the enterprise standard for multi-repo intelligence

Agents without access to an indexed codebase read files sequentially — slow, expensive, and often incomplete. Agents with an indexed codebase query by meaning — fast, targeted, and comprehensive.

---

**Trend 3: The lines between layers are blurring**

- Cursor (Layer 4) has indexing (Layer 4.5) built in
- RuFlo (Layer 5) has a pretrain indexer (Layer 4.5) built in
- GitHub Copilot (Layer 4.5) has agent capabilities (Layer 4)
- LiteLLM (Layer 3) is adding routing intelligence (Layer 5 territory)

This is healthy evolution. The layer model is a mental framework, not a rigid architecture. Use it to understand what each tool provides, not to force separation.

---

**Your stack choices — three levels:**

| Stack | What you get | When to use |
|---|---|---|
| **Layer 4 only** | Built-in agents, basic memory, partial indexing | Works for most of the things just fine, will rake up token spend|
| **Layer 4 + 4.5** | Full code intelligence, searchable codebase | Multi-file projects, any team size |
| **Layer 4 + 4.5 + 5** | Cross-tool memory, routing, safety, compounding knowledge | Multi-tool workflows, teams, long-lived projects + less token spend + best of many worlds|

### Why Add Layer 5?

Even as Layer 4 improves, external orchestration provides what no single tool can:

| Capability | Layer 4 Alone | With Layer 5 |
|---|---|---|
| **Cross-tool memory** | Each tool isolated | One memory store, all tools |
| **Model routing** | Fixed per tool | Automatic tier selection |
| **Safety gates** | Tool-dependent, inconsistent | Consistent across all tools |
| **Pattern accumulation** | Per-tool, lost on switch | Compounds across entire ecosystem |
| **Switch tools mid-task** | Lose all context | Seamless handoff via shared memory |
| **Team knowledge sharing** | Manual, ad hoc | Automatic via shared memory store |

### About This Guide Series

- **Part 1 (this doc):** Generic concepts — applies regardless of your stack choices
- **Parts 2-4:** Concrete implementation using **RuFlo** as our Layer 5 example

You may choose RuFlo, another orchestration tool, or rely solely on Layer 4's built-in features. The architectural concepts in Part 1 remain valid regardless.

---

| **No implicit codebase knowledge** | Agent doesn't know your conventions, patterns, or architecture. | It will generate code that violates your standards unless explicitly guided. |
| **Token limits** | Context windows are finite (typically 100K-200K tokens). | Large files, long conversations, or too much history causes truncation. |
| **Hallucination risk** | Agent confidently generates incorrect code, imports, or APIs. | You must review everything. The agent is not a replacement for your judgment. |
| **No execution environment** | Agent suggests commands but doesn't run them (unless explicitly allowed). | You control execution — the agent cannot accidentally delete production. |

**Key insight:** The agent is a powerful autocomplete with reasoning capabilities, not a replacement developer. Your role shifts from *writing* to *directing, reviewing, and approving*.

---

## 2. Core Concepts Developers Must Master

### Context

**What it is:** Everything the agent can "see" at this moment — the conversation history, file contents you've shared, and any tool results.

**Why it matters:** Context is finite. A 200K token window sounds large, but a single large file can consume 50K tokens. Poor context management leads to:
- Truncated history (agent forgets earlier instructions)
- Expensive token costs
- Confusion (agent references files you didn't intend)

**Best practices:**
- Share only relevant files, not entire directories
- Use memory/search instead of re-reading files
- Break large tasks into focused sessions

---

### Memory

**What it is:** Persistent storage of patterns, decisions, and conventions across sessions.

**Why it matters:** Without memory, every session starts from zero. The agent re-learns your:
- Project architecture
- Coding conventions
- Testing patterns
- Common pitfalls

**With memory (ruflo):**
```
Session 1: Agent learns your NestJS module structure → stores pattern
Session 2: Agent searches memory → retrieves pattern → implements correctly
Session 10: Agent has 50 patterns → knows your codebase better than a new hire
```

**Types of memory:**
| Type | Scope | Example |
|---|---|---|
| **Session** | One conversation | What we discussed 5 minutes ago |
| **Project** | This codebase | `horizon-grid` architecture, modules |
| **Global** | All your projects | Generic patterns (NestJS guards, React hooks) |
| **Cross-session** | Accumulated over time | Everything the agent has learned |

---

### Session

**What it is:** One continuous conversation with the agent. Starts when you open the tool, ends when you close it.

**Session lifecycle:**
```
Open tool → Submit first prompt → Agent responds → [repeat] → Close tool
         ↑___________________________________________|
                          Session
```

**Why it matters:** When the session ends, the agent's internal state is lost. Only what you explicitly store in memory persists.

---

### Cross-Session Learning

**What it is:** The compounding value of accumulated patterns across multiple sessions, days, or weeks.

**The power curve:**
```
Day 1:   Agent knows nothing about your project
Week 1:  Agent knows basic architecture
Week 4:  Agent knows common patterns and gotchas
Month 3: Agent knows codebase better than a new team member
```

**This is where ruflo delivers value.** Without it, you stay at Day 1 forever.

---

### Tool Calling

**What it is:** The agent invoking external capabilities beyond text generation.

**Common tools:**
| Tool | Purpose | Example |
|---|---|---|
| `Read` | Read file contents | Agent reads `src/auth/guard.ts` |
| `Write` | Create new files | Agent writes `src/user/service.ts` |
| `Edit` | Modify existing files | Agent updates function implementation |
| `Bash` | Execute shell commands | Agent runs `npm test` |
| `memory_search` | Search stored patterns | Agent recalls auth pattern |
| `swarm_init` | Initialize multi-agent coordination | Agent spawns specialist agents |

**Why it matters:** Tools extend the agent from a text generator to an active participant in your development workflow.

---

### Routing

**What it is:** Selecting the right model/capability for the task complexity.

**The 3-tier model:**

| Tier | Model | Cost | Best for |
|---|---|---|---|
| **1** | WASM Agent Booster | $0, <1ms | Simple transforms (var→const, add types) |
| **2** | Haiku / Sonnet | $0.0002-0.003 | Bug fixes, refactoring, implementation |
| **3** | Opus / Claude | $0.015 | Architecture, security design, complex reasoning |

**Why it matters:** Not every task needs the most expensive model. Routing saves 50-90% on token costs while maintaining quality.

**Ruflo's role:** Automatically recommends the right tier before the agent starts work.

---

## 3. Feature Development Lifecycle with Agents

Traditional development:
```
You → write code → test → commit → PR → merge
```

Agent-augmented development:
```
You → specify → agent understands → agent codes → you review → commit → PR → merge
     ↑___________↑_________________↑
           Your direction
```

### The Five Phases

```
┌─────────────────────────────────────────────────────────────────┐
│  SPEC → UNDERSTAND → CODE → TEST → RELEASE                      │
├─────────────────────────────────────────────────────────────────┤
│  Spec:       What to build — requirements, acceptance criteria  │
│  Understand: Existing codebase patterns, conventions, gotchas    │
│  Code:       Implementation following established patterns       │
│  Test:       Verification — unit, integration, e2e               │
│  Release:    PR, review, merge, deploy                          │
└─────────────────────────────────────────────────────────────────┘
```

### Phase Characteristics

| Phase | Cognitive Load | Agent Role | Model Tier | Key Capability |
|---|---|---|---|---|
| **Spec** | High (you) | Interview you, propose designs | Opus | Reasoning, architecture |
| **Understand** | Medium | Search memory, read minimal files | Sonnet | Memory retrieval |
| **Code** | Low (agent) | Generate boilerplate, follow patterns | Haiku/Sonnet | Execution |
| **Test** | Medium | Write tests, verify coverage | Sonnet | Validation |
| **Release** | Low (automation) | Git operations, CI coordination | Haiku | Coordination |

**Your role shifts by phase:**
- **Spec:** You are the architect. The agent asks questions and documents.
- **Understand:** You verify the agent's findings. The agent searches and reads.
- **Code:** You review. The agent implements.
- **Test:** You approve coverage. The agent writes tests.
- **Release:** You approve merge. The agent manages git.

---

## 4. Where Ruflo Fits

### Without Ruflo

```
Every session:
  Agent: "What's this project about?"
  You: [paste README, explain architecture]
  Agent: "What are your conventions?"
  You: [paste example files]
  Agent: [writes code that almost matches]
  You: [corrects, explains again]

Week 4, same project:
  Agent: "What's this project about?"
  You: [paste README, explain architecture — again]
```

### With Ruflo

```
Session 1:
  You: "Build auth feature"
  Agent: [searches memory — empty]
  You: [reviews, stores patterns]

Session 2:
  You: "Build billing feature"
  Agent: [searches memory — finds auth pattern, module structure, conventions]
  Agent: [implements correctly first time]

Week 4:
  Agent: [has 50 patterns, knows your codebase cold]
  You: "Refactor user module"
  Agent: [knows exactly what to do]
```

**Ruflo provides:**

| Capability | Without | With |
|---|---|---|
| Cross-session memory | ❌ None | ✅ SQLite + HNSW vector search |
| Model routing | ❌ Everything expensive | ✅ Automatic tier selection |
| Branch safety | ❌ Agent can push to main | ✅ Hard blocks on protected branches |
| Tool-agnostic | ❌ Each tool isolated | ✅ One memory, four tools |
| Pattern accumulation | ❌ Day 1 forever | ✅ Compounding knowledge |

---

## 5. Mental Model Shift

### Traditional Developer

```
Problem → Think → Write code → Test → Debug → Ship
         ↑______________↑
              90% of time
```

### Agent-Augmented Developer

```
Problem → Specify → Review agent output → Approve → Ship
         ↑_________↑
           Your value

Agent does:     Understanding, coding, testing, git operations
You do:         Direction, review, approval, complex reasoning
```

**What changes:**

| Aspect | Traditional | Agent-Augmented |
|---|---|---|
| **Time spent** | Writing implementations | Specifying and reviewing |
| **Key skill** | Typing speed, syntax knowledge | Communication, architecture |
| **Quality gate** | Your tests catching bugs | Your review catching misunderstandings |
| **Speed** | Limited by typing | Limited by review bandwidth |
| **Scale** | One file at a time | Entire features in parallel |

**The trap:** Thinking the agent replaces you. It doesn't. It amplifies you.

**The opportunity:** Focus on the high-value work (specification, architecture, review) while the agent handles implementation details.

---

## 6. Summary: Key Takeaways

1. **Agents are stateless** — memory is not automatic; you must explicitly store and retrieve patterns.

2. **Context is finite** — manage what you share; use memory instead of re-reading files.

3. **Cross-session learning compounds** — Week 4 with memory is dramatically more productive than Day 1.

4. **Tool calling extends capability** — from text generation to active development workflow participation.

5. **Routing saves money** — not every task needs the most expensive model.

6. **Your role shifts** — from writing code to directing, reviewing, and approving.

7. **Ruflo enables the shift** — by providing memory, routing, safety gates, and cross-tool consistency.

---

## Next: Part 2 — Ruflo Setup Guide

Part 2 covers the practical implementation: installing ruflo, configuring each tool (OpenCode, Claude Code, Codex, Cursor), setting up branch safety gates, and establishing the plan folder convention.
