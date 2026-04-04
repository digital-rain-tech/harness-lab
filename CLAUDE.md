# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a **read-only research repository** for analyzing Claude Code's architecture and implementation. The code is extracted TypeScript source — there is no build system, no tests, no package.json. You cannot run, build, or test anything here. The goal is purely to study agent harness patterns.

## Key Constraint

`types/message.ts` is **missing** from the source (imported by 184+ files but not present). Many type references to `Message`, `UserMessage`, `AssistantMessage`, etc. will be unresolved. Keep this in mind when tracing code paths.

## Architecture Overview

### Entrypoints (`src/entrypoints/`)
- `cli.tsx` — Main CLI bootstrap. Fast-paths for `--version` and `--dump-system-prompt`, then dynamically imports the full app. Uses `bun:bundle` feature flags for dead code elimination.
- `init.ts` — Full initialization (loaded dynamically by cli.tsx)
- `mcp.ts` — MCP server entrypoint
- `agentSdkTypes.ts` / `sdk/` — Agent SDK type definitions and schemas

### Core Abstractions
- **`Tool.ts`** — Central tool type system. Defines `ToolUseContext`, `ToolPermissionContext`, `ValidationResult`, tool input schemas. All tools implement this interface.
- **`Task.ts`** — Task lifecycle: types (`local_bash`, `local_agent`, `remote_agent`, `dream`, etc.), statuses (`pending` → `running` → `completed`/`failed`/`killed`), and `TaskContext` for abort/state management.
- **`QueryEngine.ts`** — Core query loop. Orchestrates API calls, tool execution, compaction, memory loading, permission handling, and message replay. This is the main "agent loop."
- **`commands.ts`** — Command registry. Imports all 70+ slash commands and conditionally loads feature-flagged ones (`PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `VOICE_MODE`, `DAEMON`) via `bun:bundle` feature gates.

### Tools (`src/tools/`)
Each tool is a directory with its own implementation, prompt, and constants:
- **BashTool** — Shell execution with security validation, sandbox mode, sed/path validation, destructive command warnings
- **AgentTool** — Subagent spawning with built-in agent types (general-purpose, explore, plan, claude-code-guide), agent memory, color management, worktree isolation
- **FileEditTool / FileReadTool / FileWriteTool** — File operations with image processing support
- **GrepTool / GlobTool** — Code search
- **MCPTool** — MCP server tool execution
- **EnterPlanModeTool / ExitPlanModeTool** — Plan mode workflow
- **EnterWorktreeTool / ExitWorktreeTool** — Git worktree isolation

### State Management (`src/state/`)
- `AppStateStore.ts` — Central app state with `CompletionBoundary` types for tracking completion events (bash commands, edits, denied tools)
- `store.ts` — Store implementation
- `selectors.ts` — State selectors

### Services (`src/services/`)
- **`api/`** — Anthropic API client, retry logic, usage tracking, rate limiting, prompt cache detection
- **`compact/`** — Message compaction (auto-compact, micro-compact, session memory compaction, grouping)
- **`mcp/`** — MCP client connections, OAuth, channel permissions, elicitation handling
- **`lsp/`** — LSP client/server management for code intelligence
- **`autoDream/`** — Background memory consolidation with locking
- **`plugins/`** — Plugin installation and management
- **`extractMemories/`** — Automatic memory extraction from conversations

### Hooks (`src/hooks/`)
React-style hooks for the Ink-based CLI UI. Notable patterns:
- `toolPermission/` — Permission system with coordinator, interactive, and swarm worker handlers
- `useQueueProcessor.ts` — Message queue processing
- `useTasksV2.ts` — Task management
- `useVoice.ts` / `useVimInput.ts` — Input mode handling

### Bridge (`src/bridge/`)
Remote session infrastructure — REPL bridge transport, JWT auth, polling config, session creation, inbound message/attachment handling. Enables the web-based and remote Claude Code experiences.

### Skills (`src/skills/`)
- `loadSkillsDir.ts` — Discovers and loads skill definitions from directories
- `bundled/` — Built-in skills (claude API, scheduling, config, debug, simplify, etc.)
- `mcpSkillBuilders.ts` — Builds skills from MCP server definitions

### Feature Flags
Conditional compilation via `bun:bundle` `feature()` calls: `PROACTIVE`, `KAIROS`, `KAIROS_BRIEF`, `BRIDGE_MODE`, `VOICE_MODE`, `DAEMON`, `ABLATION_BASELINE`. These gate entire command/feature imports at build time.
