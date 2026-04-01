# Harness Lab

Research repository for analyzing Claude Code's architecture, patterns, and implementation details.

## Disclaimer

This repository contains code extracted from the leaked Claude Code source for **research and educational purposes only**.

- **No proprietary claims**: We make no claims of ownership over this code
- **Academic use**: Intended for studying agent harness engineering, tool orchestration, and AI system architecture
- **Not for distribution**: Code is preserved as-is for analysis; do not use in commercial products
- **Credit**: All code belongs to the original authors at Anthropic

This project is not affiliated with, endorsed by, or maintained by Anthropic or Claude Code.

## What's Here

- `src/commands/` — CLI command implementations (~70+ commands)
- `src/tools/` — Tool definitions and executors (~40+ tools)
- `src/hooks/` — Pre/post tool use hooks
- `src/plugins/` — Plugin system
- `src/skills/` — Skill loading and bundled skills
- `src/buddy/` — Companion sprite feature
- `src/services/autoDream/` — Background memory consolidation
- `src/tasks/DreamTask/` — Dream task implementation
- `src/utils/` — Utility functions including messages.ts, messagePredicates.ts, messageQueueManager.ts
- `src/constants/messages.ts` — Message constants
- `src/components/messageActions.tsx` — Message action components
- Various other utilities, components, and state management

## Missing / Excluded Files

The following files appear to be excluded from the leak (original TS source not present):
- `types/message.ts` — Core message type definitions (imported by 184 files but not in source)
- These are likely original TypeScript files that were filtered out

## Research Focus

- Agent execution systems and tool orchestration patterns
- CLI-based AI workflow design
- Memory management and consolidation strategies
- Feature flagging and phased rollout systems
- Multi-agent coordination

## Motivation

Claude Code represents a significant engineering effort in AI agent systems. This repo exists to study how those systems work — from the tool execution pipeline to the user interface — without the noise of the original repository.

## Related

- [claw-code](https://github.com/instructkr/claw-code) — Clean-room rewrite in Python/Rust
