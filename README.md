# Claude Code — Architecture & Process Guide

> A deep-dive into how Claude Code works: the agent loop, tool layer, subagent spawning, memory system, and the harness that ties it all together.

---

## Table of Contents

1. [What is Claude Code?](#what-is-claude-code)
2. [The Agent Loop — Core Pattern](#the-agent-loop)
3. [Tool Layer](#tool-layer)
4. [Subagent Spawning](#subagent-spawning)
5. [Memory Architecture](#memory-architecture)
6. [The Full End-to-End Flow](#full-flow)
7. [Key Design Principles](#design-principles)
8. [Quickstart](#quickstart)
9. [Configuration](#configuration)
10. [Contributing](#contributing)

---

## What is Claude Code? <a name="what-is-claude-code"></a>

Claude Code is an **agentic coding harness** — not just a model, not just an IDE plugin, but a complete environment in which an AI agent can perceive, plan, act, and iterate until a coding task is done.

Stripped to its essence:

```
Claude Code = one agent loop
            + tools (bash, read, write, edit, grep, browser…)
            + on-demand skill loading
            + context compression
            + subagent spawning
            + task system with dependency graph
            + team coordination via async mailboxes
            + worktree isolation for parallel execution
            + permission governance
```

The model decides when to call tools and when to stop. The harness executes what the model asks for, manages context, and stays out of the way.

---

## The Agent Loop <a name="the-agent-loop"></a>

Every turn follows the same minimal loop:

```
User ──► messages[] ──► LLM ──► response
                                    │
                          stop_reason == "tool_use"?
                                   / \
                                 yes   no
                                  │     │
                           execute   return text
                            tools
                               │
                        append results
                               │
                        ──► messages[]   (loop back)
```

### Step by step

**1. Build context** — At the start of each turn, the harness injects:
- The contents of `CLAUDE.md` (project rules, coding standards)
- Any loaded skill files
- Compressed summaries of prior turns (if context was compacted)
- The current task state from the dependency graph

**2. LLM inference** — The assembled `messages[]` array is sent to the Claude model. The model responds with either a final text answer or one or more `tool_use` blocks.

**3. Tool dispatch** — If `stop_reason == "tool_use"`, the harness executes each requested tool, appends the results as `tool_result` blocks, and loops back to step 2.

**4. Stop** — When the model responds without a `tool_use`, the loop exits and the response is returned to the user (or parent agent).

This loop is intentionally minimal. There are no rigid decision trees, no hard-coded workflows. The model drives; the harness serves.

---

## Tool Layer <a name="tool-layer"></a>

Claude Code gives the agent a rich set of tools to operate on real systems:

| Tool | What it does |
|------|-------------|
| `Bash` | Run shell commands, scripts, git operations, package managers |
| `Read / Write / Edit` | Read file contents, write new files, apply targeted edits |
| `Glob / Grep` | Find files by pattern, search content with ripgrep |
| `Browser` | Fetch URLs, scrape pages, interact with web content |
| `Task (Spawn agent)` | Spawn a subagent with its own loop, tools, and isolated worktree |

Tools are the agent's hands. The model never directly modifies files or runs code — it requests tool calls, and the harness does the execution.

### Permission governance

Before any tool executes, the harness checks against a permission layer:
- **Read-only tools** (grep, glob, read) are freely allowed.
- **Write/destructive tools** (bash, write, edit) may require confirmation depending on the trust level and scope of the current task.
- **Network tools** (browser, fetch) are gated on explicit allowlists in sensitive environments.

---

## Subagent Spawning <a name="subagent-spawning"></a>

For complex tasks, the orchestrator spawns specialised subagents via the `Task` tool. Each subagent:

1. Gets its own isolated agent loop
2. Runs in a separate git worktree (preventing conflicts with parallel agents)
3. Has access to the same tool layer
4. Communicates results back via an **async mailbox**

### Three subagent archetypes

**Plan agent** — Decomposes a high-level goal into a dependency graph of subtasks. Produces a structured task list that the orchestrator and other agents can consume.

**Explore agent** — Reads source code, searches the codebase, gathers context. Specialised for reconnaissance — finding where things live before changes are made.

**Task agent** — Implements a specific change: writes code, runs tests, commits results. Works in an isolated worktree so multiple task agents can run in parallel without stepping on each other.

### Async coordination

Agents communicate via mailboxes rather than blocking calls. The orchestrator sends work to subagents and continues (or waits). When a subagent finishes, it posts its result to the orchestrator's mailbox. This enables genuine parallelism across large tasks.

```
Orchestrator
    │
    ├──[Task tool]──► Plan agent   → posts result to mailbox
    ├──[Task tool]──► Explore agent → posts result to mailbox
    └──[Task tool]──► Task agent   → posts result to mailbox
                           ↓
              Orchestrator reads mailbox, synthesises, responds
```

---

## Memory Architecture <a name="memory-architecture"></a>

Claude Code has no persistent memory in the model weights for your project — instead it uses a layered external memory system:

### Layer 1 — In-context memory (volatile)

The live `messages[]` array. Everything the model can "see" right now. Auto-compacted (summarised) when it approaches the token limit. Completely reset between sessions unless explicitly restored.

### Layer 2 — External file memory (session-scoped)

- **`CLAUDE.md`** — A markdown file at the project root. Claude reads this at the start of every session. Put your coding standards, architecture decisions, common commands, and project conventions here. This is the primary way to give Claude durable project knowledge.
- **`~/.claude/`** — User-level preferences and global instructions, loaded on startup.
- **Skill files** — Markdown files that describe how to use a tool or perform a task. Loaded on demand when the agent determines they're relevant.

### Layer 3 — Task state memory (run-scoped)

- **Dependency graph** — The task system tracks which subtasks are pending, in-progress, blocked, or done. Survives agent hops within a session.
- **Async mailboxes** — Per-agent inboxes that buffer results across parallel workstreams.
- **Git checkpoints** — Worktree commits serve as implicit state snapshots for parallel task agents.

### Layer 4 — Long-term external memory (persistent across sessions)

Via MCP (Model Context Protocol) servers:
- Vector stores / RAG (e.g., ChromaDB, pgvector) for semantic search over codebase history
- Session restore tools that ingest prior Claude Code session logs
- Arbitrary databases via MCP tool integrations

### Read order per turn

```
Long-term MCP stores
       ↓
External files (CLAUDE.md, skills)
       ↓
Task state (dep. graph, mailboxes)
       ↓
Inject into in-context window for this turn
```

---

## The Full End-to-End Flow <a name="full-flow"></a>

Here is the complete journey from a user request to a committed code change:

```
┌─────────────────────────────────────────────────────────────────┐
│  1. USER types a task in the terminal                           │
│     e.g. "Add pagination to the /users endpoint"                │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│  2. HARNESS builds context                                      │
│     • Loads CLAUDE.md, skill files, memory                      │
│     • Constructs messages[] with system prompt + task           │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│  3. ORCHESTRATOR (primary agent loop)                           │
│     • Calls LLM → receives plan + tool_use blocks               │
│     • May spawn Plan agent to build dependency graph            │
│     • May spawn Explore agents to read relevant files           │
│     • Spawns Task agents for each implementation subtask        │
└────────────────────────┬────────────────────────────────────────┘
                         │ (parallel)
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     Plan agent    Explore agent   Task agent(s)
     builds dep.   reads code,     implement in
     graph         finds patterns  isolated worktrees
          │              │              │
          └──────────────┴──────────────┘
                         │ results → mailboxes
┌────────────────────────▼────────────────────────────────────────┐
│  4. ORCHESTRATOR synthesises results                            │
│     • Reads mailboxes, checks task graph                        │
│     • Runs integration tests via Bash tool                      │
│     • Compresses context if needed                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│  5. OUTPUT returned to user                                     │
│     • Summary of changes made                                   │
│     • Files edited, tests passed, git diff                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Design Principles <a name="design-principles"></a>

**Model drives, harness serves.** Claude Code does not impose rigid workflows or second-guess the model with elaborate decision trees. It provides tools, knowledge, and context — then gets out of the way.

**Isolation by default.** Parallel task agents work in separate git worktrees. A broken change in one agent cannot corrupt another agent's work.

**Context is precious.** The harness actively manages the context window — compressing history, loading skills on demand, and using external memory to avoid blowing the token budget on stale information.

**Permission boundaries, not walls.** Destructive tools require confirmation rather than being blocked outright. The agent can always ask the user when it's uncertain.

**Stateless model, stateful harness.** The LLM has no memory between turns. The harness is responsible for reconstructing the world for each call — injecting the right context so the model can act as if it remembers.

---

## Quickstart <a name="quickstart"></a>

### Prerequisites

- Node.js 18+
- An Anthropic API key

### Install

```bash
npm install -g @anthropic-ai/claude-code
```

### Run

```bash
cd your-project
claude
```

Claude Code will read your `CLAUDE.md` (if present), load project context, and enter the interactive loop.

### Add project memory

Create a `CLAUDE.md` at your project root:

```markdown
# Project: My App

## Architecture
- Backend: FastAPI (Python 3.11)
- Database: PostgreSQL via SQLAlchemy
- Tests: pytest, run with `make test`

## Conventions
- All endpoints go in `src/api/`
- DB models go in `src/models/`
- Never commit directly to `main` — always use feature branches

## Common commands
- Start dev server: `make dev`
- Run linter: `make lint`
- Apply migrations: `alembic upgrade head`
```

The richer your `CLAUDE.md`, the more autonomously Claude Code can work without asking clarifying questions.

---

## Configuration <a name="configuration"></a>

| File | Scope | Purpose |
|------|-------|---------|
| `CLAUDE.md` | Project | Coding standards, architecture, conventions |
| `~/.claude/CLAUDE.md` | User | Personal preferences across all projects |
| `.claude/settings.json` | Project | Tool permissions, model selection, MCP servers |
| `~/.claude/settings.json` | User | Global defaults |

### MCP server configuration

Connect long-term memory, internal tools, and external APIs via MCP:

```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/project"]
    }
  }
}
```

---

## Contributing <a name="contributing"></a>

Issues and pull requests welcome. When contributing:

1. Fork the repo and create a feature branch
2. Add or update tests for any changed behaviour
3. Update `CLAUDE.md` if you change project conventions
4. Open a PR — a description of *why* the change matters helps reviewers move faster

---

*Built on the Claude API by Anthropic. This repository is an independent harness implementation and educational reference — not the official Anthropic Claude Code product.*
