# ai-dev-setup

Reference docs for AI-assisted development tooling setup.

## Contents

| File | What it covers |
|---|---|
| `ruflo-daily-workflow.md` | Day-to-day development lifecycle with RuFlo as memory/orchestration layer |
| `ruflo-mcp-integration-plan.md` | Full installation guide: what problem ruflo solves, config for all 4 tools, SDLC phase breakdown, observations |

## Tools covered

- **OpenCode** — primary coding agent (VS Code extension)
- **Claude Code** — terminal-based agent
- **Codex** — headless/automated tasks
- **Cursor** — visual IDE with Roo-Cline extension

## Quick start

```bash
# Verify ruflo is running
ruflo --version
ruflo memory stats

# Search project memory
ruflo memory search --query "your question about the codebase"

# Check daily metrics
ruflo hooks metrics --v3-dashboard
```
