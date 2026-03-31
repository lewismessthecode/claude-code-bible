# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Claude Code is Anthropic's CLI for interacting with Claude from the terminal. It's a TypeScript application (~1,900 files, 512K+ lines) running on **Bun**, using **React + Ink** for terminal UI, **Commander.js** for CLI parsing, and **Zod v4** for schema validation.

This repository is a source snapshot (no package.json, tsconfig, or test files). The code lives entirely under `src/`.

## Architecture

### Core Execution Loop

The fundamental flow is: user input -> `QueryEngine.ts` -> Anthropic API -> streaming response -> tool execution -> loop until `end_turn`.

- **`main.tsx`** - CLI entry point. Runs parallel prefetch (MDM settings, keychain, API preconnect) as side effects *before* heavy imports for startup optimization.
- **`QueryEngine.ts`** - Core LLM query loop. Manages token budgets, tool-call orchestration, streaming, retry logic, and auto-compaction.
- **`query.ts`** - Query pipeline and tool orchestration layer between QueryEngine and the API.
- **`Tool.ts`** - Base type definitions for all tools (input schemas, permission models, execution interface).
- **`tools.ts`** - Tool registry. Registers all 42+ tools with conditional loading via feature flags and `process.env.USER_TYPE`.
- **`commands.ts`** - Slash command registry (~50+ commands). Same conditional loading pattern.

### Key Subsystems

| Directory | Purpose |
|---|---|
| `tools/` | Self-contained tool implementations (BashTool, FileReadTool, AgentTool, etc.) |
| `commands/` | Slash command implementations (`/commit`, `/review`, `/plan`, etc.) |
| `services/` | External integrations (API client, MCP, OAuth, LSP, analytics, compaction) |
| `components/` | React/Ink UI components (~140 files) |
| `screens/` | Full-screen UIs (REPL.tsx is the main interactive session - 874K) |
| `hooks/` | React hooks, notably `toolPermission/` for the permission system |
| `state/` | App state management (Zustand-like store + React Context) |
| `bridge/` | IDE extension communication (VS Code, JetBrains) |
| `coordinator/` | Multi-agent orchestration (feature-gated) |
| `skills/` | Skill system (bundled + user-defined prompt workflows) |
| `plugins/` | Plugin system (bundled + user-installed) |
| `tasks/` | Task management subsystem (local, remote, in-process agents) |
| `utils/` | 330+ utility files organized by subdomain |
| `entrypoints/` | Initialization logic, SDK entry points |
| `bootstrap/` | Global singleton state (session ID, CWD, telemetry) - **leaf module, no src/ imports** |

### Permission System (3 layers)

1. **Filesystem rules** (`utils/permissions/filesystem.ts`) - glob-based path allow/deny, worktree boundaries
2. **Shell rules** (`utils/permissions/bashPermissions.ts`) - command whitelisting, destructive operation detection
3. **Prompt-time evaluation** (`hooks/useCanUseTool.tsx`) - permission modes: `default`, `plan`, `auto`, `bypassPermissions`

### Feature Flags & Dead Code Elimination

Bun's `bun:bundle` `feature()` function strips code at build time. Ant-only code uses `process.env.USER_TYPE === 'ant'`.

Key flags: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `VOICE_MODE`, `AGENT_TRIGGERS`, `COORDINATOR_MODE`, `DAEMON`, `HISTORY_SNIP`

### Multi-Agent System

- `AgentTool` spawns sub-agents with isolated token budgets
- `TeamCreateTool`/`TeamDeleteTool` manage agent teams
- `SendMessageTool` enables async inter-agent messaging
- Team state stored in `~/.claude/teams/{team}/`

### Message Flow

```
UserMessage -> normalizeMessagesForAPI() -> POST /messages
  -> StreamingResponse -> ContentBlocks (text + tool_use)
  -> findToolByName() -> useCanUseTool() permission check
  -> Tool.execute() -> ToolResultBlockParam
  -> attach result to next API request -> loop
  -> stop_reason: end_turn -> done
```

### State Architecture

- **Bootstrap state** (`bootstrap/state.ts`): Global singleton, no imports from src/ (prevents circular deps). Holds session ID, CWD, telemetry refs.
- **App state** (`state/AppState.tsx`): Interactive UI state via Zustand-like store + React Context. Messages, agents, permissions, notifications.

### Startup Optimization Pattern

`main.tsx` fires side effects (MDM subprocess, keychain reads, API preconnect) *before* import statements so they run in parallel with ~135ms of module evaluation:

```typescript
profileCheckpoint('main_tsx_entry')
startMdmRawRead()        // fire subprocess, don't wait
startKeychainPrefetch()  // queue async keychain reads
// heavy imports follow - they evaluate while prefetch runs
```

### Circular Dependency Pattern

Lazy `require()` calls break circular dependencies (tools.ts -> TeamCreateTool -> ... -> tools.ts):

```typescript
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
```

### Notable Large Files

| File | Description |
|---|---|
| `screens/REPL.tsx` (874K) | Main interactive session UI |
| `services/api/claude.ts` (122K) | Anthropic API client |
| `services/mcp/client.ts` (116K) | MCP server lifecycle |
| `bridge/bridgeMain.ts` (112K) | IDE bridge main loop |
| `tools/BashTool/bashPermissions.ts` (96K) | Bash permission rules |

### Service Layer Key Files

- `services/api/claude.ts` - Anthropic SDK wrapper, message batching, usage tracking
- `services/mcp/client.ts` - MCP server connections, resource discovery, tool integration
- `services/compact/` - Context compression and auto-compaction
- `services/analytics/growthbook.ts` - Feature flags and A/B experiments
- `services/lsp/` - Language Server Protocol for code intelligence

### Import Conventions

- Import markers (`// ANT-ONLY`) must not be reordered (biome organize-imports disabled for these files)
- Feature-gated imports use `require()` inside conditionals for DCE
- `.js` extensions required in all import paths (Bun ESM resolution)
