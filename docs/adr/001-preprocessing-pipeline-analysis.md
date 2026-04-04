# ADR 001: Claude Code Preprocessing Pipeline Analysis

**Status:** Published
**Date:** 2026-04-04

## Context

This repository exists to study Claude Code's agent harness architecture. The preprocessing pipeline — everything that happens between user input and the API call — is the most architecturally significant subsystem. This ADR documents the pipeline's structure, key design decisions, and patterns worth studying.

## Findings

### Pipeline Overview

User input passes through 6 stages before the model sees it:

1. **System Prompt Assembly** (`src/constants/prompts.ts`)
2. **User Context Memoization** (`src/context.ts`)
3. **Input Processing** (`src/utils/processUserInput/processUserInput.ts`)
4. **Hook Execution** (`src/utils/hooks.ts`)
5. **Attachment Injection** (`src/utils/attachments.ts`)
6. **Message Normalization** (`src/utils/messages.ts:normalizeMessagesForAPI`)

### Stage 1: System Prompt Assembly

**Key file:** `src/constants/prompts.ts`

The system prompt is built from ~15 sections split by `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`:

- **Static prefix** (globally cacheable): identity, system rules, task guidelines, action safety, tool usage, tone/style, output efficiency
- **Dynamic suffix** (per-session): session guidance, memory, env info, language, MCP instructions, scratchpad, function-result-clearing

**Design decision: Cache boundary marker.** Everything before the boundary gets `scope: 'global'` for prompt prefix caching across orgs. This is a cost/latency optimization at scale.

**Design decision: Conditional sections via `USER_TYPE`.** Anthropic internal users (`ant`) get different prompt sections:
- Stricter comment-writing rules
- False-claims mitigation
- Numeric length anchors (≤25 words between tool calls)
- Verification agent requirements for non-trivial implementations
- Assertiveness counterweights

These are annotated with `@[MODEL LAUNCH]` comments indicating per-model-generation tuning.

**Design decision: Feature flags via `bun:bundle`.** Entire sections are DCE'd (dead-code-eliminated) at build time: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `VOICE_MODE`, `ULTRAPLAN`, `HISTORY_SNIP`, `COORDINATOR_MODE`, `TOKEN_BUDGET`, `EXPERIMENTAL_SKILL_SEARCH`, `VERIFICATION_AGENT`, `BUDDY`.

### Stage 2: User Context

**Key file:** `src/context.ts`

Two memoized functions computed once per conversation:

- **`getUserContext()`** — Walks directory hierarchy for CLAUDE.md files. Result cached in side channel for `yoloClassifier.ts` (auto-mode permission classifier) to avoid import cycles. Injects current date.
- **`getSystemContext()`** — Git snapshot: branch, default branch, user name, short status (truncated at 2K chars), last 5 commits. Includes optional cache-breaking injection (`BREAK_CACHE_COMMAND` feature flag).

**Design decision: Memoization, not live updates.** Git status is frozen at conversation start. The model is told explicitly: "this status is a snapshot in time, and will not update during the conversation." Changed files between turns are handled by the attachment system instead.

### Stage 3: Input Processing

**Key file:** `src/utils/processUserInput/processUserInput.ts`

Processing order:

1. **Image resizing** — `maybeResizeAndDownsampleImageBlock()` for API limits. Metadata collected for hidden `isMeta` message.
2. **Bridge safety** — Remote inputs filtered. Only "bridge-safe" commands allowed. Unknown `/foo` falls through as text.
3. **Ultraplan keyword** — Special keyword in pre-expansion input rewrites to `/ultraplan` command. Feature-flagged, interactive-only.
4. **Slash command routing** — 70+ commands registered in `src/commands.ts`.
5. **Attachment extraction** — `@`-mentioned files, MCP resources, IDE selections resolved.

**Design decision: Pre-expansion keyword detection.** Ultraplan keyword is detected on the *pre-expansion* input (before `[Pasted text #N]` expansion) so pasted content can't accidentally trigger it.

### Stage 4: Hook Execution

**Key files:** `src/utils/hooks.ts`, `src/utils/hooks/hookHelpers.ts`

`UserPromptSubmit` hooks are user-defined shell commands that can:
- **Block** the prompt (returns system warning, erases original input)
- **Prevent continuation** (stops processing, keeps prompt in context)
- **Inject additional context** (truncated at `MAX_HOOK_OUTPUT_LENGTH = 10000` chars)

**Design decision: Hooks as extension points.** Same pattern as Git hooks — lets users add CI/CD-style guardrails (linting, policy checks) without code changes.

### Stage 5: Attachment System

**Key file:** `src/utils/attachments.ts`

The most complex stage. ~25 attachment types computed in **parallel** via `Promise.all`:

**User input attachments** (run first):
- `at_mentioned_files` — reads `@path/to/file` references
- `mcp_resources` — resolves MCP resource URIs
- `agent_mentions` — detects `@agent-<name>` syntax
- `skill_discovery` — Haiku-based skill search (feature-flagged)

**Per-turn context attachments** (run after user input attachments):
- `changed_files` — files modified since last turn
- `nested_memory` — CLAUDE.md from newly-accessed directories
- `deferred_tools_delta` — tools added/removed since last turn
- `agent_listing_delta` — agent definition changes
- `mcp_instructions_delta` — MCP server connect/disconnect
- `todo_reminders` / `task_reminders` — incomplete task reminders
- `plan_mode` / `plan_mode_exit` — plan mode state
- `auto_mode` / `auto_mode_exit` — auto-mode state (feature-flagged)
- `teammate_mailbox` — inter-agent messages (swarm mode)
- `team_context` — shared team state (swarm mode)
- `companion_intro` — buddy sprite context (feature-flagged)
- `date_change` — calendar date rollover notification
- `ultrathink_effort` — thinking effort adjustment
- `compaction_reminder` — context limit warnings (feature-flagged)
- `context_efficiency` — snip-based efficiency metrics (feature-flagged)
- `queued_commands` — agent-scoped command queue drain
- `dynamic_skill` / `skill_listing` — skill metadata
- `critical_system_reminder` — high-priority system state

**Design decision: Ordering dependency.** User input attachments run first because `@`-mentioned files populate `nestedMemoryAttachmentTriggers`, which `nested_memory` then reads. This ensures CLAUDE.md files from newly-referenced directories are loaded.

**Design decision: Deterministic RAG.** Rather than vector-based retrieval, context injection is rule-based and purpose-built per attachment type. Trade-off: no semantic similarity search, but fully deterministic and debuggable.

**Design decision: 1-second timeout.** Attachment computation has a hard 1-second abort timeout to prevent blocking the user.

### Stage 6: Message Normalization

**Key file:** `src/utils/messages.ts` (`normalizeMessagesForAPI`)

Transformations before the API call:

1. **Attachment reordering** — bubbles attachments up past user messages until hitting tool_result or assistant message
2. **Virtual message stripping** — removes display-only messages (`isVirtual`)
3. **Error-based content stripping** — maps API errors (PDF too large, image too large) to block types to strip from preceding user messages
4. **Consecutive user message merging** — required for Bedrock compatibility
5. **Tool reference cleanup** — strips `tool_reference` blocks for disconnected MCP servers
6. **System message conversion** — `local_command` system messages become user messages

## Key Patterns

### Pattern: Cache Boundary Architecture
Explicit markers separate cacheable from dynamic content. Cache hit rates directly impact cost at scale.

### Pattern: Attachment-Based Context Engineering
Instead of traditional RAG, use parallel deterministic functions to inject relevant context per turn. Debuggable, predictable, no embedding drift.

### Pattern: Feature Flag Dead Code Elimination
`bun:bundle` `feature()` calls enable compile-time DCE. Entire features (commands, attachments, prompt sections) are eliminated from external builds.

### Pattern: Memoized Snapshots + Delta Attachments
Heavy context (git status, CLAUDE.md) is computed once. Changes between turns are tracked via lightweight delta attachments (`changed_files`, `deferred_tools_delta`).

### Pattern: Hooks as Platform Extension
User-defined shell commands at lifecycle points (prompt submit, tool use, session start) make the agent extensible without code changes.

## Key Files

| File | Role |
|---|---|
| `src/constants/prompts.ts` | System prompt assembly with cache boundary |
| `src/context.ts` | Memoized user/system context (CLAUDE.md, git) |
| `src/utils/processUserInput/processUserInput.ts` | Input processing pipeline |
| `src/utils/hooks.ts` | Hook execution system |
| `src/utils/attachments.ts` | Attachment injection (~25 types) |
| `src/utils/messages.ts` | Message normalization for API |
| `src/QueryEngine.ts` | Orchestrates the full query lifecycle |
| `src/constants/systemPromptSections.ts` | Section registry for prompt assembly |
| `src/utils/queryContext.ts` | System prompt parts fetcher |
| `src/utils/queryHelpers.ts` | Message normalization helpers |
