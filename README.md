# Claude-code

`Claude-code` is a large TypeScript/Bun source snapshot for an interactive coding assistant that combines a terminal UI, command-driven workflows, tool execution, Model Context Protocol (MCP) integration, remote bridge/session plumbing, and multi-agent orchestration.

This repository is useful as a recovery/reference codebase today, but it is not a complete, self-contained application checkout yet.

## Current Status

As currently imported, this repository contains the application source tree itself rather than the full original project scaffold.

What is present:

- core entrypoints such as `main.tsx`, `commands.ts`, `Tool.ts`, `tools.ts`, `QueryEngine.ts`, and `Task.ts`
- a large command surface under `commands/`
- a large tool surface under `tools/`
- UI, services, state, bridge, plugin, skill, and swarm-related modules
- roughly 4,100 tracked files in total

What is not present at the repo root:

- `package.json`
- lockfiles such as `pnpm-lock.yaml`, `yarn.lock`, or `package-lock.json`
- top-level TypeScript/build config files from the original project scaffold
- CI/release metadata
- license metadata

Practical implication:

- this repo is good for code reading, indexing, search, selective extraction, and reconstruction work
- it is not guaranteed to build or run out of the box until the missing project scaffolding is restored

## What This Codebase Appears To Do

Based on the included source, this project implements an agentic coding assistant with:

- an interactive CLI/TUI application bootstrapped from `main.tsx`
- a large slash-command style command system registered in `commands.ts`
- a typed tool execution model centered around `Tool.ts` and `tools.ts`
- built-in tools for shell execution, file operations, search, web access, notebooks, tasks, planning, MCP resources, and agent workflows
- plugin and skill discovery/loading
- remote session and bridge infrastructure
- multi-agent and swarm/coordinator utilities
- React/Ink-based terminal UI components and dialogs
- supporting systems for config, auth, telemetry, permissions, keybindings, onboarding, vim-like interaction, and voice-related features

Many capabilities are feature-gated. The source uses `bun:bundle` feature flags and environment checks, so the exact command and tool surface depends on the original build configuration.

## Repository Layout

One important detail: the imported code now lives at the repository root. In the original project, some imports were clearly relying on alias/path configuration such as `src/...`. That means the current layout is best understood as a recovered source root, not a finished standalone repo.

High-level map:

| Path | Purpose |
| --- | --- |
| `main.tsx` | Main startup path, bootstrap logic, CLI wiring, session initialization |
| `commands.ts` | Central command registry and feature-gated command loading |
| `Tool.ts` | Shared tool contracts, context, permissions, and tool-related types |
| `tools.ts` | Source of truth for built-in tool registration |
| `commands/` | 100+ command modules covering config, review, agents, MCP, skills, plugins, files, status, usage, export, and more |
| `tools/` | Built-in tools such as bash, file read/edit/write, grep/glob, web fetch/search, plan/worktree, MCP, tasks, and agent/team operations |
| `components/` and `screens/` | React/Ink UI components and interactive views |
| `state/` and `context/` | App state, providers, and cross-cutting runtime context |
| `services/` | Higher-level service integrations including MCP, APIs, policy, tips, and background infrastructure |
| `bridge/`, `remote/`, and `server/` | Remote control, session bridge, transport, and server-side plumbing |
| `plugins/` and `skills/` | Extensibility systems for plugin and skill loading |
| `coordinator/`, `tasks/`, and `utils/swarm/` | Multi-agent coordination, teammate workflows, and swarm execution helpers |
| `migrations/` | Configuration and behavior migrations across versions |
| `schemas/`, `types/`, and `constants/` | Shared contracts and configuration types |
| `voice/`, `buddy/`, `vim/`, `keybindings/` | Optional UX surfaces and interaction modes |

## Notable Capabilities Visible In Source

The current source tree points to a broad feature set, including:

- command-driven workflows for setup, config, review, export, onboarding, usage, status, files, tasks, permissions, skills, plugins, and MCP
- tool execution for shell commands, file manipulation, notebook editing, grep/glob search, task output, LSP, and web access
- MCP resource listing/reading and tool discovery
- plan mode and worktree-oriented workflows
- agent/team/swap-in coordination logic and teammate-style execution backends
- bridge and remote session management
- terminal UI components for onboarding, dialogs, history, search, approval flows, and diagnostics
- optional or gated surfaces for voice, browser, Chrome, desktop, mobile, stickers, and buddy/assistant behavior

Representative command areas visible from the registry include:

- `help`, `config`, `login`, `logout`, `status`, `usage`, `export`
- `review`, `security-review`, `diff`, `files`, `branch`, `commit-push-pr`
- `agents`, `tasks`, `plan`, `permissions`, `skills`, `plugin`, `reload-plugins`
- `mcp`, `teleport`, `remote-env`, `desktop`, `mobile`, `ide`
- `theme`, `vim`, `keybindings`, `voice`, `summary`, `release-notes`

Representative built-in tools visible from `tools.ts` include:

- `BashTool`
- `FileReadTool`, `FileEditTool`, `FileWriteTool`
- `GlobTool`, `GrepTool`
- `WebFetchTool`, `WebSearchTool`
- `NotebookEditTool`
- `AgentTool`, `SkillTool`
- task-oriented tools such as `TaskCreateTool`, `TaskGetTool`, `TaskUpdateTool`, `TaskListTool`
- MCP-related tools such as `ListMcpResourcesTool`, `ReadMcpResourceTool`, and `ToolSearchTool`
- planning/worktree tools and additional gated tools enabled by build flags

## Important Entry Points

If you are trying to understand or reconstruct the project, these are the best places to start:

1. `main.tsx`
   Main bootstrap path. It wires startup profiling, config loading, auth/session setup, CLI parsing, telemetry initialization, feature gating, and REPL/session launch behavior.
2. `commands.ts`
   Central registry for builtin commands. This is the best high-level view of the command surface and which features are conditionally enabled.
3. `Tool.ts`
   Defines the core tool execution contracts, permission context, progress types, and runtime interfaces shared across tool implementations.
4. `tools.ts`
   Registers the builtin tools and shows which tools are always present versus feature-gated or environment-specific.
5. `components/App.tsx`
   Thin but important top-level wrapper for interactive sessions, wiring the app state, stats, and FPS-related providers used by the terminal UI.

## Working With This Repository

Right now, the safest way to treat this repo is as a source archive that you can explore and incrementally restore.

Recommended workflows:

- use it for code search, reference, and architectural recovery
- extract specific subsystems into other projects if needed
- reconstruct missing root metadata before attempting a full local build

Useful exploration commands:

```bash
rg "feature\\(" /shared/Claude-code
rg "import .* from 'src/" /shared/Claude-code
rg "Tool" /shared/Claude-code/tools /shared/Claude-code/tools.ts
rg "commands/" /shared/Claude-code/commands.ts
```

## What Is Missing To Make It Runnable

To turn this repository into a normal buildable project, you will likely need to restore or recreate:

1. the package manifest and lockfile
2. the original TypeScript path alias configuration
3. Bun/Node build scripts
4. linting, formatting, and test configuration
5. CI workflows and release metadata
6. any generated assets or build-time codegen inputs not included in this snapshot

The code strongly suggests Bun-based builds plus a TypeScript/React/Ink stack, but without the original root manifests it is not possible to state the exact install or run commands confidently.

## Caveats

- Some imports still assume the original project layout and alias setup.
- Some features are only enabled when specific build flags or environment variables are present.
- Because the repo is source-only, setup instructions from the original project are not yet recoverable from this checkout alone.
- Before sharing or operationalizing the code more broadly, review secrets, endpoints, credentials, and licensing posture carefully.

## Suggested Next Steps

If you want this repository to become a polished standalone project, the highest-value follow-up work is:

1. add missing root project metadata (`package.json`, lockfile, tsconfig, formatter/linter config)
2. decide whether to keep the source at repo root or move it back under a conventional `src/` layout
3. document the command surface and tool model in dedicated docs
4. add a license and contribution policy
5. add tests and CI once the build is reproducible

Until then, this README should be read as documentation for a recovered source tree rather than a finished distributable repository.
