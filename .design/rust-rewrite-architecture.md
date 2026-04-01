# Feature: Rust Rewrite — Workspace Architecture & Implementation Plan

## Summary
Rewrite Claude Code from TypeScript/Bun (~1,900 files, ~512K LOC) to Rust, organized as a Cargo workspace with 21 crates. The rewrite preserves exact feature parity — every tool (~54 in `getAllBaseTools()` + dynamic MCP tools), command (~103 total: ~75 public + ~28 internal-only), skill (17 bundled), permission mode, protocol, file format, and UI behavior must be reproduced identically. This document defines the crate structure, dependency graph, module mapping, cross-cutting concerns (analytics, migration), crate dependency choices, and phased implementation timeline.

## Requirements

- REQ-1: The Cargo workspace must contain exactly 22 crates matching the dependency graph below, with no circular dependencies between crates
- REQ-2: Every TypeScript source directory under `src/` (35 top-level directories, 18 root-level orchestration files) must map to exactly one Rust crate per the module inventory table
- REQ-3: All ~54 tool implementations registered in `src/tools.ts` `getAllBaseTools()` must have corresponding Rust implementations in `claude-tools`, including feature-gated tools (LSPTool, PowerShellTool, ScheduleCronTool, CronDeleteTool, CronListTool, SleepTool, RemoteTriggerTool, MonitorTool, SendUserFileTool, PushNotificationTool, SubscribePRTool, WebBrowserTool, SnipTool, ListPeersTool, WorkflowTool, OverflowTestTool, CtxInspectTool, TerminalCaptureTool, SuggestBackgroundPRTool, VerifyPlanExecutionTool, etc.) plus dynamically-registered MCP tools (MCPTool, McpAuthTool, SyntheticOutputTool)
- REQ-4: All ~103 commands registered in `src/commands.ts` must have corresponding Rust implementations in `claude-commands` — ~75 public commands in `COMMANDS()` plus ~28 in `INTERNAL_ONLY_COMMANDS` (backfillSessions, breakCache, bughunter, commit, commitPushPr, ctx_viz, goodClaude, issue, mockLimits, bridgeKick, version, resetLimits, onboarding, share, summary, teleport, antTrace, perfIssue, env, oauthRefresh, debugToolCall, etc.)
- REQ-5: All 17 bundled skills must be embedded in `claude-skills`: batch, claudeApi (+ claudeApiContent), claudeInChrome, debug, dream, hunter, keybindings, loop, loremIpsum, remember, runSkillGenerator, scheduleRemoteAgents, simplify, skillify, stuck, updateConfig, verify (+ verifyContent) — with feature gates for dream (KAIROS_DREAM), hunter (REVIEW_ARTIFACT), and runSkillGenerator (RUN_SKILL_GENERATOR)
- REQ-6: Feature flag gating must use Rust conditional compilation (`#[cfg(feature = "...")]`) to replicate all 29+ dynamic feature gates from `bun:bundle`: `kairos`, `proactive`, `agent-triggers`, `agent-triggers-remote`, `monitor-tool`, `kairos-push-notification`, `kairos-github-webhooks`, `bridge-mode`, `daemon`, `voice-mode`, `history-snip`, `workflow-scripts`, `ccr-remote-setup`, `experimental-skill-search`, `ultraplan`, `torch`, `uds-inbox`, `fork-subagent`, `buddy`, `overflow-test-tool`, `context-collapse`, `terminal-panel`, `web-browser-tool`, `coordinator-mode`, `building-claude-apps`, `review-artifact`, `run-skill-generator`, `kairos-brief`, `kairos-dream`, `mcp-skills`
- REQ-7: The build must produce a single binary with no external runtime dependency (replacing the Bun runtime). Static linking where possible: `rustls-tls` for TLS (no `native-tls` / system OpenSSL dependency), `git2` with `vendored` feature for libgit2, `keyring` with file-backend fallback for environments without system secret storage
- REQ-8: Analytics and telemetry (OpenTelemetry metrics with 8 counter types — session, LOC, PR, commit, cost, token, codeEditToolDecision, activeTime — plus GrowthBook feature flags, event logging with Datadog sink) must be implemented in `claude-analytics` with lazy initialization matching the TypeScript lazy-import pattern
- REQ-9: All 11 migration functions (from `migrate_auto_updates_to_settings` through `migrate_fennec_to_opus`) must be preserved in `claude-config` with identical version numbering (CURRENT_MIGRATION_VERSION = 11)
- REQ-10: File format compatibility must be maintained for all 10 configuration/storage formats (global config, settings, project config, session transcripts, memory files, keybindings, MCP config, skills, hooks, task lists) at their existing paths
- REQ-11: The Rust crate dependency table must use the prescribed crate for each purpose (reqwest for HTTP, tokio-tungstenite for WebSocket, ratatui+crossterm for TUI, taffy for flexbox layout, syntect for syntax highlighting, similar for diff, etc.)
- REQ-12: The implementation must follow 6 phases over 24 weeks, with each phase producing a testable milestone that can be validated against the TypeScript version

## Acceptance Criteria

- [ ] AC-1: `cargo build --release` compiles the entire workspace with zero errors and zero warnings on stable Rust >= 1.85 (Edition 2024)
- [ ] AC-2: `cargo test --workspace` passes with no failures, covering at minimum the same test scenarios as the TypeScript test suite
- [ ] AC-3: The dependency graph between crates matches the specified DAG — verified by `cargo tree` showing no circular dependencies and correct parent-child relationships
- [ ] AC-4: Running `claude --version` from the Rust binary produces equivalent output to the TypeScript version
- [ ] AC-5: Every tool listed in the TypeScript `tools.ts` `getAllBaseTools()` has a corresponding `impl Tool for ...` in `claude-tools` — verified by a test under `--all-features` that compares tool name sets, plus a matrix test verifying each feature flag enables/disables the correct tool subset
- [ ] AC-6: Every command listed in the TypeScript `commands.ts` (both `COMMANDS()` and `INTERNAL_ONLY_COMMANDS`) has a corresponding entry in the Rust command registry in `claude-commands` — verified by a test under `--all-features` that compares command name sets
- [ ] AC-7: Session transcripts written by the TypeScript version can be loaded by the Rust version via `--resume` / `--continue` without error
- [ ] AC-8: Configuration files (`.claude.json`, `~/.claude/settings.json`, `.mcp.json`, `~/.claude/keybindings.json`) written by TypeScript are parsed correctly by Rust — verified by round-trip tests
- [ ] AC-9: Feature-gated code is correctly excluded from the binary when features are disabled — verified by checking binary symbols or running with feature combinations
- [ ] AC-10: Each implementation phase produces a binary that passes its phase-specific integration tests before proceeding to the next phase
- [ ] AC-11: The Rust binary startup time is equal to or faster than the TypeScript/Bun version — measured by `hyperfine` benchmarking `claude --version`
- [ ] AC-12: Memory usage under steady-state REPL operation is equal to or lower than the TypeScript version — measured by `/proc/self/status` VmRSS comparison

## Architecture

### Crate Structure

```
claude-code/
├── Cargo.toml                    (workspace root, edition = "2024")
├── crates/
│   ├── claude-cli/               (binary crate — entry point, CLI parsing, TUI bootstrap)
│   ├── claude-commands/          (library crate — ~103 command implementations)
│   ├── claude-core/              (query engine, tool loop, message types, coordinator)
│   ├── claude-tools/             (all ~54 tool implementations + dynamic MCP wrappers)
│   ├── claude-state/             (SessionState 98 fields + ~210 exports, AppState, Store)
│   ├── claude-auth/              (OAuth PKCE, API keys, keychain, JWT, multi-provider)
│   ├── claude-bridge/            (remote control, CCR v1/v2, WebSocket, assistant, server)
│   ├── claude-mcp/               (MCP client, 4 transports, tool bridging)
│   ├── claude-plugins/           (plugin loader, marketplace, builtin plugins)
│   ├── claude-skills/            (skill loading, 17 bundled skills, frontmatter parsing)
│   ├── claude-config/            (settings hierarchy, global/project config, 11 migrations)
│   ├── claude-memory/            (auto-memory, team memory, scanning, selection)
│   ├── claude-hook-engine/       (hook execution engine + schemas: command, prompt, agent, http)
│   ├── claude-tasks/             (background tasks: shell, agent, remote, teammate, dream)
│   ├── claude-tui/               (ratatui components, layout, buddy, keybindings, screens)
│   ├── claude-native/            (color-diff via syntect, file-index fuzzy search)
│   ├── claude-analytics/         (event logging, GrowthBook, OpenTelemetry, Datadog sink)
│   ├── claude-net/               (proxy, mTLS, preconnect, upstream CONNECT-over-WS relay)
│   ├── claude-utils/             (shared: fs, git, path, shell, permissions, 90+ modules)
│   ├── claude-vim/               (vim mode state machine: motions, operators, text objects)
│   ├── claude-query/             (query config, deps, stop hooks, token budget)
│   └── claude-types/             (shared type definitions: Message, Command, Permission, etc.)
```

### Dependency Graph

```
claude-cli
  ├── claude-commands
  │   ├── claude-core
  │   ├── claude-state
  │   ├── claude-config
  │   ├── claude-auth
  │   └── claude-types
  ├── claude-core
  │   ├── claude-tools
  │   │   ├── claude-types
  │   │   ├── claude-state          (tools access SessionState directly)
  │   │   └── claude-mcp            (MCP wrapper tools: McpAuthTool, ListMcpResources, etc.)
  │   ├── claude-state
  │   │   └── claude-types
  │   ├── claude-auth
  │   ├── claude-mcp [optional]     (injected via trait objects; not required for Phase 2)
  │   ├── claude-hook-engine [optional]  (injected via trait objects; not required for Phase 2)
  │   └── claude-query
  ├── claude-tui
  │   ├── claude-native
  │   ├── claude-state
  │   ├── claude-types
  │   └── claude-vim
  ├── claude-bridge
  │   ├── claude-auth
  │   └── claude-net
  ├── claude-plugins
  │   └── claude-types
  ├── claude-skills
  │   └── claude-types
  ├── claude-config
  │   └── claude-types
  ├── claude-memory
  │   └── claude-types
  ├── claude-tasks
  ├── claude-analytics
  │   └── claude-types
  ├── claude-hook-engine
  │   └── claude-types
  ├── claude-net
  └── claude-utils
```

Note: `claude-core`'s dependencies on `claude-mcp` and `claude-hook-engine` are behind optional Cargo features. In Phase 2, these are injected as trait objects via `Box<dyn HookRunner>` and `Box<dyn McpToolProvider>`, allowing the core engine to function without them. Phase 3 provides the concrete implementations.

### Module Inventory (TypeScript Source → Rust Crate)

| TypeScript Directory | Files | LOC | Rust Crate | Notes |
|---|---|---|---|---|
| `src/` (root files) | 18 | ~6,000 | `claude-cli`, `claude-core` | Entry points, query engine, tool/command registries |
| `src/assistant/` | 1 | ~150 | `claude-bridge` | Session history pagination for remote sessions |
| `src/bootstrap/` | 1 | ~1,759 | `claude-state` | SessionState singleton (98 fields, ~210 exports) |
| `src/bridge/` | 31 | ~8,000 | `claude-bridge` | Remote Control, CCR v1/v2, WebSocket, JWT, polling |
| `src/buddy/` | 6 | ~1,298 | `claude-tui` | Procedural companion (deterministic PRNG) |
| `src/cli/` | 15 | ~10,000 | `claude-cli` | Print orchestrator, StructuredIO, transports, auth handlers |
| `src/commands/` | 189 | ~25,000 | `claude-commands` | ~103 slash commands (local, local-jsx, prompt types) |
| `src/components/` | 389 | ~82,000 | `claude-tui` | Terminal UI components (ratatui widgets) |
| `src/constants/` | 21 | ~5,000 | `claude-core`, `claude-config` | OAuth, betas, prompts, tool limits, figures |
| `src/context/` | 9 | ~1,000 | `claude-tui` | UI contexts (notifications, voice, stats) |
| `src/coordinator/` | 1 | ~370 | `claude-core` | Coordinator mode (multi-agent orchestration) |
| `src/entrypoints/` | 8 | ~3,750 | `claude-cli` | CLI bootstrap, init, MCP server, SDK types |
| `src/hooks/` | 85 | ~15,000 | `claude-tui` | UI hooks (permissions, input, IDE, remote) |
| `src/ink/` | 40+ | ~20,000 | `claude-tui` | Custom terminal renderer → replaced by ratatui |
| `src/keybindings/` | 14 | ~3,500 | `claude-tui` | Keybinding system (17 contexts, 60+ actions, chords) |
| `src/memdir/` | 8 | ~2,500 | `claude-memory` | Memory system (prompts, paths, scanning, selection) |
| `src/migrations/` | 11 | ~1,500 | `claude-config` | Data migrations v1-v11 |
| `src/native-ts/` | 4 | ~4,500 | `claude-native` | Color diff (syntect), file index (nucleo scoring) |
| `src/outputStyles/` | 1 | ~200 | `claude-config` | Output style loading from markdown |
| `src/plugins/` | 2 | ~180 | `claude-plugins` | Built-in plugin registry |
| `src/query/` | 4 | ~650 | `claude-query` | Query config, deps, stop hooks, token budget |
| `src/remote/` | 4 | ~1,130 | `claude-bridge` | Remote session manager, WebSocket, SDK adapter |
| `src/schemas/` | 1 | ~220 | `claude-hook-engine` | Hook Zod schemas (command, prompt, agent, http) |
| `src/screens/` | 3 | ~6,000 | `claude-tui` | REPL (5K lines), ResumeConversation, Doctor |
| `src/server/` | 3 | ~360 | `claude-bridge` | Direct-connect session manager |
| `src/services/oauth/` | ~10 | ~3,000 | `claude-auth` | OAuth client, token management, refresh |
| `src/services/api/` | ~8 | ~4,000 | `claude-core` | Anthropic SDK wrapper, streaming, retry |
| `src/services/analytics/` | ~6 | ~2,500 | `claude-analytics` | GrowthBook, event logging, Datadog |
| `src/services/mcp/` | ~12 | ~5,000 | `claude-mcp` | MCP client, registry, validation, server approval |
| `src/services/compact/` | ~4 | ~2,000 | `claude-core` | Context compression, summarization |
| `src/services/lsp/` | ~5 | ~3,000 | `claude-tools` | LSP integration for LSPTool |
| `src/services/plugins/` | ~8 | ~3,500 | `claude-plugins` | Plugin loading, marketplace, versioning |
| `src/services/tools/` | ~6 | ~2,500 | `claude-core` | Tool execution orchestration, hooks |
| `src/services/{SessionMemory,extractMemories,teamMemorySync}/` | ~8 | ~3,000 | `claude-memory` | Memory extraction, sync, session memory |
| `src/services/{settingsSync,remoteManagedSettings,policyLimits}/` | ~10 | ~4,000 | `claude-config` | Settings sync, managed settings, policy |
| `src/services/{remaining: tips, notifier, diagnostics, voice, etc.}` | ~50 | ~7,500 | `claude-tui`, `claude-core` | UI services, rate limiting, diagnostics |
| `src/skills/` | 20 | ~8,000 | `claude-skills` | Skill loading, 17 bundled skills |
| `src/state/` | 6 | ~1,190 | `claude-state` | Store, AppState, selectors, onChange |
| `src/tasks/` | 12 | ~3,700 | `claude-tasks` | Shell, agent, remote, teammate, dream tasks |
| `src/tools/` | 100+ | ~30,000 | `claude-tools` | ~54 tool implementations + dynamic MCP wrappers |
| `src/types/` | 11 | ~3,000 | `claude-types` | Message, command, permission, plugin, hook types |
| `src/upstreamproxy/` | 2 | ~800 | `claude-net` | CCR upstream CONNECT-over-WebSocket proxy |
| `src/utils/` | 90+ | ~30,000 | `claude-utils` | Auth, config, git, path, shell, permissions |
| `src/vim/` | 5 | ~1,520 | `claude-vim` | Vim state machine (motions, operators, text objects) |
| `src/voice/` | 1 | ~80 | `claude-tui` | Voice mode feature gate |

**Total: ~1,900 files, ~512K lines of TypeScript → 22 Rust crates**

### Rust Crate Dependencies

| Purpose | TypeScript Library | Rust Crate |
|---|---|---|
| HTTP client | axios | `reqwest` (with rustls-tls) |
| WebSocket | ws / global WebSocket | `tokio-tungstenite` |
| JSON | JSON.parse/stringify | `serde_json` |
| Schema validation | zod | `serde` + `validator` |
| CLI parsing | Commander.js | `clap` (derive API) |
| Terminal UI | React + Ink | `ratatui` + `crossterm` |
| Flexbox layout | Yoga (custom port) | `taffy` |
| Syntax highlighting | highlight.js | `syntect` |
| Diff | diff (npm) | `similar` |
| OAuth | custom | `oauth2` |
| UUID | crypto.randomUUID | `uuid` (v4) |
| Regex | RegExp | `regex` |
| Async runtime | Bun event loop | `tokio` (multi-thread) |
| Channels | N/A | `tokio::sync::mpsc`, `broadcast` |
| File watching | chokidar | `notify` |
| Glob patterns | ripgrep (external) | `globset` + `ignore` |
| Content search | ripgrep (external) | `grep-regex` + `grep-searcher` (ripgrep library crates) |
| Git operations | git CLI | `tokio::process::Command` (shell out to `git` CLI, matching TS) |
| Shell execution | child_process | `tokio::process::Command` |
| Secure storage | macOS security binary | `keyring` |
| mTLS | Node TLS | `rustls` + `native-tls` (platform fallback) |
| Proxy | https-proxy-agent | `reqwest` built-in proxy |
| Markdown parsing | custom | `pulldown-cmark` |
| YAML parsing | yaml (npm) | `serde_yml` (successor to archived `serde_yaml`) |
| Grapheme handling | Intl.Segmenter | `unicode-segmentation` |
| Semver | semver (npm) | `semver` |
| Protobuf (upstream proxy) | hand-encoded | `prost` |
| SSE | fetch + ReadableStream | `reqwest-eventsource` |
| Cron parsing | custom | `cron` |
| Fuzzy matching | Fuse.js + custom | `nucleo` |
| QR code | qrcode (npm) | `qrcode` |
| Process management | child_process | `nix` (Unix signals) |
| File locking | proper-lockfile | `fd-lock` (replaces unmaintained `fs2`) |
| Base64 | Buffer | `base64` |
| SHA256 | crypto | `sha2` |
| String width | custom stringWidth | `unicode-width` |
| Color output | chalk | ANSI escape codes (direct) |
| OpenTelemetry | @opentelemetry/* | `opentelemetry` + `opentelemetry-otlp` |
| Error handling | N/A (exceptions) | `thiserror` + `anyhow` |
| Serialization derive | N/A | `serde` (derive) |
| Tracing/logging | console.log | `tracing` + `tracing-subscriber` |

### Error Handling Strategy

Each crate defines typed error enums via `thiserror`:

```rust
// claude-auth/src/error.rs
#[derive(Debug, thiserror::Error)]
pub enum AuthError {
    #[error("OAuth flow failed: {0}")]
    OAuthFailed(String),
    #[error("token expired and refresh failed")]
    TokenRefreshFailed,
    #[error("keychain access denied: {0}")]
    KeychainError(#[from] keyring::Error),
    // ...
}
```

**Rules:**
- Library crates (`claude-core`, `claude-tools`, `claude-state`, etc.) use `thiserror` for typed, matchable error enums
- Only the binary crate (`claude-cli`) uses `anyhow` for top-level error reporting to the user
- Error conversion at crate boundaries: each crate's error type implements `From<DependencyError>` for upstream errors it can encounter
- Tool execution returns `Result<ToolResult, ToolError>` (typed), not `anyhow::Result` — the tool loop needs to match on error variants for retry/permission/abort decisions
- TypeScript `try/catch` with untyped exceptions maps to `Result<T, E>` propagation via `?` operator — panics are reserved for invariant violations that indicate bugs, never for recoverable errors
- Fatal errors (OOM, stack overflow) bubble to a unified top-level handler in `claude-cli` that logs to `inMemoryErrorLog` before exiting

### Cross-Cutting Concerns

**Analytics & Telemetry** (`claude-analytics`):
- Event logging with pre-sink queuing (events fire before sink attachment at startup)
- GrowthBook feature flag client with cached + blocking evaluation modes
- OpenTelemetry integration: Meter, Counter (7 metric types: session, LOC, PR, commit, cost, token, activeTime), LoggerProvider, TracerProvider
- Lazy initialization via `once_cell::sync::Lazy` to match TypeScript's lazy `import()` pattern

**Migration & Compatibility** (`claude-config`):
- Sequential migration runner (v1 through v11) with version tracking in `~/.claude.json`
- Migrations: settings restructuring, model name updates (sonnet-1m→sonnet-45→sonnet-46, opus→opus-1m, fennec→opus), permission mode changes, remote control defaults
- All existing file formats remain readable: JSON configs, JSONL transcripts, Markdown+YAML frontmatter memories/skills, JSON keybindings

**Feature Flag Compilation**:
- TypeScript uses `bun:bundle` feature flags for dead code elimination
- Rust equivalent: Cargo features in `Cargo.toml` + `#[cfg(feature = "...")]` attributes
- Key features: `kairos`, `bridge-mode`, `buddy`, `workflow-scripts`, `voice-mode`, `agent-triggers`, `ultraplan`, `internal-only`

### Design Principles

1. **Feature parity** — Every login method, tool, command, permission mode, and UI behavior preserved exactly
2. **Same file/folder layout** — Configuration files, skill directories, memory paths, and session storage at identical paths
3. **Same protocols** — OAuth flows, WebSocket connections, MCP stdio/SSE/HTTP transports, CCR v1/v2 all preserved
4. **Same prompts** — System prompts, tool descriptions, and all user-facing text reproduced verbatim (54K bytes in `src/constants/prompts.ts`)
5. **Prompt cache boundary** — `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` position preserved to maintain API prompt cache hit rates

### Implementation Phases

**Phase 1: Foundation (Weeks 1-4)**
- `claude-types`: Shared type definitions (Message, Tool trait, Command, Permission enums)
- `claude-utils`: File operations, path handling, git utilities, shell execution (90+ modules)
- `claude-config`: Settings loading (6-level hierarchy), global config, project config, 11 migrations (all named: migrate_auto_updates_to_settings, migrate_bypass_permissions_accepted_to_settings, migrate_enable_all_project_mcp_servers_to_settings, reset_pro_to_opus_default, migrate_sonnet_1m_to_sonnet_45, migrate_legacy_opus_to_current, migrate_sonnet_45_to_sonnet_46, migrate_opus_to_opus_1m, migrate_repl_bridge_enabled_to_remote_control_at_startup, reset_auto_mode_opt_in_for_default_offer, migrate_fennec_to_opus)
- `claude-auth`: OAuth PKCE flow, API key management, secure storage, multi-provider (FirstParty, Bedrock, Vertex, Foundry)
- `claude-state`: Store pattern, SessionState (98 fields, ~210 exports), basic AppState
- **Milestone**: Can load configuration, authenticate, and maintain state

**Phase 2: Core Engine (Weeks 5-8)**
- `claude-core`: Message types, query engine, streaming architecture, tool loop — with trait-object injection points for `HookRunner` and `McpToolProvider` (concrete implementations deferred to Phase 3)
- `claude-tools`: File tools (Read, Edit, Write, Glob, Grep, Notebook), BashTool with security analysis
- `claude-net`: Proxy configuration, mTLS, HTTP client creation, NO_PROXY parsing
- `claude-analytics`: Event logging, GrowthBook client, OpenTelemetry 8 counters
- `claude-query`: Query config, dependencies, stop hooks, token budget tracking
- **Milestone**: Can execute a headless query with file and bash tools (hooks and MCP bypassed via no-op trait impls)

**Phase 3: Tool Ecosystem (Weeks 9-12)**
- `claude-tools`: Remaining tools (Web, Task, Agent, LSP, Config, Worktree, MCP wrappers, Cron, Remote — completing all ~54 tools)
- `claude-mcp`: MCP client with all 4 transports (Stdio, SSE, HTTP, WebSocket), tool bridging — provides concrete `McpToolProvider` impl
- `claude-hook-engine`: Hook execution engine (command, prompt, agent, http event types) — provides concrete `HookRunner` impl
- `claude-memory`: Memory system with 4 types (User, Feedback, Project, Reference), scanning, team memory security (two-pass path validation)
- **Milestone**: Full tool ecosystem operational, hooks active, MCP servers connectable

**Phase 4: UI & Interaction (Weeks 13-16)**
- `claude-tui`: Ratatui widgets (message list, prompt input, permission dialogs, scroll, spinner), virtual scrolling
- `claude-vim`: Vim state machine (motions: h/l/j/k/w/b/e/W/B/E/0/^/$, operators: d/c/y, text objects: 16 pairs, find: f/F/t/T)
- `claude-native`: Color diff (syntect + similar + grapheme-aware wrapping), file index (nucleo-equivalent scoring)
- `claude-commands`: ~103 command implementations
- `claude-cli`: REPL screen (~5K lines equivalent), keybinding system (17 contexts, 60+ actions)
- **Milestone**: Interactive REPL fully functional

**Phase 5: Advanced Features (Weeks 17-20)**
- `claude-bridge`: Remote control, CCR v1/v2, 3 transport types (WebSocket, Hybrid, SSE), JWT refresh scheduler, reconnection with exponential backoff
- `claude-tasks`: All task types (LocalShell, LocalAgent, RemoteAgent, InProcessTeammate, Dream, LocalWorkflow, MonitorMcp)
- `claude-plugins`: Plugin loader, marketplace integration, builtin plugins, manifest validation
- `claude-skills`: Skill loading from 5 sources (managed, user, project, additional dirs, legacy), 17 bundled skills, frontmatter parsing, conditional activation
- **Milestone**: Bridge mode and plugin ecosystem operational

**Phase 6: Integration & Polish (Weeks 21-24)**
- Full integration testing: Rust binary vs TypeScript version side-by-side
- Session resume compatibility (TypeScript JSONL transcripts loadable by Rust)
- Performance benchmarking: startup time (`hyperfine`), steady-state memory (VmRSS), tool execution latency
- Edge case hardening: Unicode (grapheme clusters, NFC normalization), large files (>100MB), network errors (timeout, retry, proxy)
- System prompt verification: byte-for-byte comparison of generated prompts
- Feature flag matrix testing: verify each of 29+ feature flags enables/disables correct tool/command subset under `--all-features` canonical build
- **Milestone**: Production-ready binary passing full compatibility test suite

## Open Questions

### Q1: Voice mode architecture
**Decision**: Defer voice mode to post-launch. The TypeScript source has only ~80 lines of feature gate code in `src/voice/`, with the bulk in hooks/services that depend on external STT/TTS services. Voice will be added as a follow-up after core feature parity is achieved.

### Q2: git2 vs git CLI for git operations
**Decision**: Use `git` CLI for all git operations via `tokio::process::Command`. This matches the TypeScript implementation exactly (which shells out for everything), avoids `git2`'s known limitations (no sparse checkout, limited worktree support, different merge behavior), eliminates the libgit2 build dependency, and guarantees identical behavior since the same `git` binary is invoked. The `git2` crate is removed from the dependency table.

### Q3: Internal-only features in open source build
**Decision**: Compile all commands into the binary and gate at runtime via `USER_TYPE` environment variable, matching the current TypeScript behavior. This avoids requiring separate builds for internal vs external, simplifies CI, and preserves the existing deployment model. Internal commands check `std::env::var("USER_TYPE") == Ok("ant")` at registration time.

## Out of Scope

- Mobile/tablet client (this is a terminal CLI rewrite only)
- New features not present in the TypeScript version (the goal is exact parity, not enhancement)
- Web UI (claude.ai/code frontend) — this document covers only the CLI binary
- Python/Node SDK wrappers — the Agent SDK types are preserved but SDK implementations are separate projects
- Infrastructure changes (API endpoints, OAuth server, MCP proxy) — only the client is rewritten
