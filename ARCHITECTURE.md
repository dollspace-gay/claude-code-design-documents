# Claude Code — Architecture

Claude Code is Anthropic's agentic CLI for interacting with Claude from the terminal. Written in TypeScript, runs on Bun. It streams LLM responses, executes tools (file edits, shell commands, web searches, sub-agents), and manages permissions, context budgets, and persistent memory across sessions.

## System Overview

```
 ┌──────────────────────────────────────────────────────────────────────┐
 │                         USER / IDE                                  │
 │              terminal  ·  VS Code  ·  JetBrains                     │
 └──────────┬──────────────────────────────────────┬───────────────────┘
            │ stdin / CLI args                     │ bridge protocol
            ▼                                      ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │  ENTRY  (main.tsx)                                                  │
 │  Commander.js arg parsing · React/Ink renderer bootstrap            │
 │  parallel prefetch: MDM settings, keychain, GrowthBook flags        │
 └──────────────────────────┬──────────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │  REPL.tsx  │  │  CLI.tsx   │  │  bridge/   │
     │ interactive│  │ one-shot   │  │ IDE ext.   │
     └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
           └────────────────┼───────────────┘
                            ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │  QUERY ORCHESTRATION  (query.ts)                                    │
 │  load context (git, CLAUDE.md, memories) · build system prompt      │
 │  normalize messages & attachments · manage conversation turns       │
 └──────────────────────────┬──────────────────────────────────────────┘
                            ▼
 ┌──────────────────────────────────────────────────────────────────────┐
 │  QUERY ENGINE  (QueryEngine.ts)                                     │
 │  Anthropic SDK streaming call · parse tool_use blocks               │
 │  token counting · context compaction · budget tracking              │
 │                                                                      │
 │  ┌─────────────────────────────────────────────────────────────┐    │
 │  │  TOOL LOOP  (loops until stop_reason != tool_use)           │    │
 │  │                                                             │    │
 │  │   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐  │    │
 │  │   │  Permission  │───▶│  Tool Exec   │───▶│  Result     │  │    │
 │  │   │  Check       │    │  (parallel)  │    │  Handling   │  │    │
 │  │   └──────────────┘    └──────┬───────┘    └─────────────┘  │    │
 │  │                              │                              │    │
 │  │          ┌───────────────────┼──────────────────┐           │    │
 │  │          ▼                   ▼                   ▼          │    │
 │  │   ┌────────────┐     ┌────────────┐      ┌───────────┐    │    │
 │  │   │  BashTool  │     │ FileEdit   │      │ AgentTool │    │    │
 │  │   │  Grep/Glob │     │ Read/Write │      │ (recurse) │    │    │
 │  │   │  WebSearch │     │ Notebook   │      │ MCPTool   │    │    │
 │  │   └────────────┘     └────────────┘      └───────────┘    │    │
 │  └─────────────────────────────────────────────────────────────┘    │
 └──────────────────────────┬──────────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
 ┌────────────────┐ ┌──────────────┐ ┌──────────────────┐
 │  State         │ │  Context     │ │  Rendering       │
 │  AppState tree │ │  Compaction  │ │  Ink components  │
 │  (immutable)   │ │  (auto-trim) │ │  streamed output │
 └────────────────┘ └──────────────┘ └──────────────────┘

 ── supporting services ───────────────────────────────────────────────

 ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐
 │  OAuth   │ │  MCP     │ │  Plugins │ │  Skills  │ │  Hooks      │
 │  keychain│ │  servers │ │  loader  │ │  system  │ │  pre/post   │
 │  JWT     │ │  + tools │ │  + mktpl │ │  bundled │ │  query/tool │
 └──────────┘ └──────────┘ └──────────┘ └──────────┘ └─────────────┘
```

## Major Modules

| Module | Path | Responsibility |
|---|---|---|
| **Entry** | `main.tsx`, `setup.ts`, `entrypoints/` | CLI parsing, bootstrap, auth, telemetry init |
| **Query** | `query.ts`, `QueryEngine.ts` | LLM request loop, streaming, tool dispatch, compaction |
| **Tools** | `Tool.ts`, `tools/` (~44 tools) | Agent capabilities: Bash, file ops, search, web, sub-agents |
| **Commands** | `commands.ts`, `commands/` (~88 cmds) | Slash commands (`/commit`, `/review`, `/help`, etc.) |
| **Components** | `components/` (~140 files) | React/Ink terminal UI — REPL, dialogs, tool output rendering |
| **State** | `state/AppState.tsx`, `state/store.ts` | Immutable session state tree with listener-based updates |
| **Hooks** | `hooks/` (~100 files) | React hooks for permission checks, UI logic, state binding |
| **Permissions** | `hooks/toolPermission/`, `utils/permissions/` | Multi-layer security: modes, rules, bash validation |
| **Services/API** | `services/api/` | Anthropic SDK wrapper, streaming, retry, usage tracking |
| **Services/MCP** | `services/mcp/` | Model Context Protocol client, tool schema translation |
| **Services/Compact** | `services/compact/` | Context compression when approaching token budget |
| **Plugins** | `services/plugins/`, `utils/plugins/` | Plugin discovery, versioning, loading, marketplace |
| **Skills** | `skills/` | Reusable workflows (bundled + user-defined) |
| **Coordinator** | `coordinator/` | Multi-agent orchestration and swarm management |
| **Memory** | `memdir/` | Persistent memory extraction and auto-save across sessions |
| **Tasks** | `tasks/` | Background task tracking and shell spawning |
| **Bridge** | `bridge/` | IDE extension protocol (VS Code, JetBrains) |
| **Analytics** | `services/analytics/` | GrowthBook feature flags, OpenTelemetry, event logging |

## Data Flow

1. **Input** — User types in the REPL, passes CLI args, or sends via IDE bridge
2. **Context loading** — Git status, CLAUDE.md files, memories, and settings are gathered
3. **System prompt assembly** — Tools, skills, commands, and permissions are bundled into the prompt
4. **LLM streaming** — Request sent to Anthropic API via SDK; response streamed back
5. **Tool loop** — Each `tool_use` block is permission-checked, executed (possibly in parallel), and its result fed back to the model. Sub-agents recurse through the same query path
6. **Compaction** — When token usage approaches the budget, older context is compressed
7. **Rendering** — Text and tool results are rendered to the terminal via Ink components
8. **Persistence** — Session transcripts saved; memories extracted for future recall

## Key Design Patterns

- **Parallel prefetch** — MDM, keychain, and feature flag reads fire concurrently at startup before heavy module evaluation
- **Immutable state** — `AppState` is `DeepImmutable`; updates create new objects and notify listeners
- **Streaming tool execution** — `StreamingToolExecutor` runs multiple tools in parallel with cancellation via `AbortController`
- **Dead code elimination** — `bun:bundle` feature flags strip unused modules at build time
- **Permission layering** — Mode (`default`/`plan`/`auto`/`bypass`) + per-tool allow/deny rules + bash-specific command/path validation
- **Agent recursion** — `AgentTool` creates isolated sub-agent contexts with their own state and budget, calling `query()` recursively
- **Lazy loading** — Heavy dependencies (OpenTelemetry, gRPC, coach logic) are `import()`-ed at first use

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Bun |
| Language | TypeScript (strict) |
| Terminal UI | React + Ink |
| CLI parsing | Commander.js |
| LLM client | @anthropic-ai/sdk |
| Protocol | @modelcontextprotocol/sdk |
| Validation | Zod v4 |
| Search | ripgrep (external binary) |
| Telemetry | OpenTelemetry + gRPC (lazy) |
| Feature flags | GrowthBook |
