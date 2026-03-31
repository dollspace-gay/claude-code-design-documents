# Claude Code — Rust Rewrite Design Document

This document is a comprehensive design specification for rewriting Claude Code (Anthropic's agentic CLI) from TypeScript/Bun to Rust, preserving all existing functionality exactly.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Crate Structure](#2-crate-structure)
3. [Module Inventory](#3-module-inventory)
4. [Core Types & Traits](#4-core-types--traits)
5. [Authentication & OAuth](#5-authentication--oauth)
6. [State Management](#6-state-management)
7. [Query Engine & Streaming](#7-query-engine--streaming)
8. [Tool System](#8-tool-system)
9. [Terminal UI](#9-terminal-ui)
10. [Bridge & Remote Sessions](#10-bridge--remote-sessions)
11. [MCP Integration](#11-mcp-integration)
12. [Plugin & Skill Systems](#12-plugin--skill-systems)
13. [Configuration & Settings](#13-configuration--settings)
14. [Memory System](#14-memory-system)
15. [Task & Agent Orchestration](#15-task--agent-orchestration)
16. [CLI Entry Points & Commands](#16-cli-entry-points--commands)
17. [Networking & Proxy](#17-networking--proxy)
18. [Native Modules](#18-native-modules)
19. [Keybindings](#19-keybindings)
20. [Vim Mode](#20-vim-mode)
21. [Analytics & Telemetry](#21-analytics--telemetry)
22. [Migration & Compatibility](#22-migration--compatibility)
23. [Rust Crate Dependencies](#23-rust-crate-dependencies)
24. [Implementation Phases](#24-implementation-phases)

---

## 1. System Overview

Claude Code is a ~512K-line TypeScript CLI running on Bun. It provides an interactive REPL where users converse with Claude, which can execute tools (bash, file edits, web search, sub-agents) to perform software engineering tasks.

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                      USER / IDE                                 │
 │             terminal  ·  VS Code  ·  JetBrains                  │
 └──────────┬─────────────────────────────────┬────────────────────┘
            │ stdin / CLI args                │ bridge protocol
            ▼                                 ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │  ENTRY  (main.rs)                                               │
 │  clap arg parsing · TUI bootstrap                               │
 │  parallel prefetch: MDM settings, keychain, feature flags       │
 └──────────────────────────┬──────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │    REPL    │  │  One-shot  │  │   Bridge   │
     │ interactive│  │  headless  │  │  IDE ext.  │
     └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
           └────────────────┼───────────────┘
                            ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │  QUERY ORCHESTRATION                                            │
 │  load context (git, CLAUDE.md, memories) · build system prompt  │
 │  normalize messages & attachments · manage conversation turns   │
 └──────────────────────────┬──────────────────────────────────────┘
                            ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │  QUERY ENGINE                                                   │
 │  Anthropic SDK streaming · parse tool_use blocks                │
 │  token counting · context compaction · budget tracking          │
 │                                                                  │
 │  ┌──────────────────────────────────────────────────────────┐   │
 │  │  TOOL LOOP  (loops until stop_reason != tool_use)        │   │
 │  │                                                          │   │
 │  │   Permission ──▶ Tool Exec ──▶ Result Handling           │   │
 │  │      Check       (parallel)                              │   │
 │  │                      │                                   │   │
 │  │      ┌───────────────┼──────────────────┐                │   │
 │  │      ▼               ▼                   ▼               │   │
 │  │  BashTool      FileEdit/Read/Write   AgentTool           │   │
 │  │  Grep/Glob     Notebook              MCPTool             │   │
 │  │  WebSearch     WebFetch              SkillTool           │   │
 │  └──────────────────────────────────────────────────────────┘   │
 └──────────────────────────┬──────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
 ┌────────────────┐ ┌──────────────┐ ┌──────────────────┐
 │  State         │ │  Context     │ │  Rendering       │
 │  AppState      │ │  Compaction  │ │  TUI (ratatui)   │
 │  (Arc<Mutex>)  │ │  (auto-trim) │ │  streamed output │
 └────────────────┘ └──────────────┘ └──────────────────┘

 ── supporting services ──────────────────────────────────────────

 ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
 │  OAuth   │ │  MCP     │ │  Plugins │ │  Skills  │ │  Hooks   │
 │  keychain│ │  servers │ │  loader  │ │  system  │ │  pre/post│
 │  JWT     │ │  + tools │ │  + mktpl │ │  bundled │ │  query   │
 └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

### Key Design Principles for the Rewrite

1. **Feature parity** — Every login method, tool, command, permission mode, and UI behavior must be preserved exactly
2. **Same file/folder layout** — Configuration files, skill directories, memory paths, and session storage remain identical
3. **Same protocols** — OAuth flows, WebSocket connections, MCP stdio/SSE/HTTP transports, CCR v1/v2 all preserved
4. **Same prompts** — System prompts, tool descriptions, and all user-facing text reproduced verbatim

---

## 2. Crate Structure

```
claude-code/
├── Cargo.toml                    (workspace root)
├── crates/
│   ├── claude-cli/               (binary crate — entry point, CLI parsing, TUI)
│   ├── claude-core/              (query engine, tool loop, message types)
│   ├── claude-tools/             (all 40+ tool implementations)
│   ├── claude-state/             (AppState, Store, selectors)
│   ├── claude-auth/              (OAuth, API keys, keychain, JWT)
│   ├── claude-bridge/            (remote control, CCR, WebSocket transports)
│   ├── claude-mcp/               (MCP client, transports, tool bridging)
│   ├── claude-plugins/           (plugin loader, marketplace, builtin plugins)
│   ├── claude-skills/            (skill loading, bundled skills, frontmatter)
│   ├── claude-config/            (settings, global config, project config)
│   ├── claude-memory/            (auto-memory, team memory, memory scanning)
│   ├── claude-hooks/             (hook execution: command, prompt, agent, http)
│   ├── claude-tasks/             (background tasks, shell, agent, remote, dream)
│   ├── claude-tui/               (terminal UI: ratatui components, layout)
│   ├── claude-native/            (color-diff, file-index, yoga-layout)
│   ├── claude-analytics/         (event logging, GrowthBook, telemetry)
│   ├── claude-net/               (proxy, mTLS, preconnect, upstream proxy)
│   ├── claude-utils/             (shared utilities, fs, git, path, shell)
│   └── claude-vim/               (vim mode state machine)
```

### Dependency Graph (simplified)

```
claude-cli
  ├── claude-core
  │   ├── claude-tools
  │   ├── claude-state
  │   ├── claude-auth
  │   ├── claude-mcp
  │   └── claude-hooks
  ├── claude-tui
  │   └── claude-native (color-diff, yoga)
  ├── claude-bridge
  ├── claude-plugins
  ├── claude-skills
  ├── claude-config
  ├── claude-memory
  ├── claude-tasks
  ├── claude-analytics
  ├── claude-net
  ├── claude-utils
  └── claude-vim
```

---

## 3. Module Inventory

### Source Directory → Rust Crate Mapping

| TypeScript Directory | Files | LOC | Rust Crate | Notes |
|---|---|---|---|---|
| `src/` (loose files) | 18 | ~6,000 | `claude-cli`, `claude-core` | Entry points, query engine, tool/command registries |
| `src/assistant/` | 1 | ~150 | `claude-bridge` | Session history pagination for remote sessions |
| `src/bootstrap/` | 1 | ~1,758 | `claude-state` | Global session state (212 fields, 120+ accessors) |
| `src/bridge/` | 31 | ~8,000 | `claude-bridge` | Remote Control, CCR v1/v2, WebSocket, JWT, polling |
| `src/buddy/` | 6 | ~1,298 | `claude-tui` | Procedural companion (deterministic PRNG) |
| `src/cli/` | 15 | ~10,000 | `claude-cli` | Print orchestrator, StructuredIO, transports, auth handlers |
| `src/commands/` | 189 | ~25,000 | `claude-cli` | 80+ slash commands (local, local-jsx, prompt types) |
| `src/components/` | 389 | ~82,000 | `claude-tui` | React/Ink terminal UI components |
| `src/constants/` | 21 | ~5,000 | `claude-core`, `claude-config` | OAuth, betas, prompts, tool limits, figures |
| `src/context/` | 9 | ~1,000 | `claude-tui` | React contexts (notifications, voice, stats) |
| `src/coordinator/` | 1 | ~370 | `claude-core` | Coordinator mode (multi-agent orchestration) |
| `src/entrypoints/` | 8 | ~3,750 | `claude-cli` | CLI bootstrap, init, MCP server, SDK types |
| `src/hooks/` | 83 | ~15,000 | `claude-tui` | React hooks (permissions, input, IDE, remote) |
| `src/ink/` | 40+ | ~20,000 | `claude-tui` | Custom terminal renderer (React reconciler + Yoga) |
| `src/keybindings/` | 14 | ~3,500 | `claude-tui` | Keybinding system (schema, parser, resolver, chords) |
| `src/memdir/` | 8 | ~2,500 | `claude-memory` | Memory system (prompts, paths, scanning, selection) |
| `src/migrations/` | 11 | ~1,500 | `claude-config` | Data migrations (settings, model aliases, permissions) |
| `src/moreright/` | 1 | ~30 | `claude-tui` | External build stub |
| `src/native-ts/` | 4 | ~4,500 | `claude-native` | Color diff, file index (nucleo), yoga layout |
| `src/outputStyles/` | 1 | ~200 | `claude-config` | Output style loading from markdown |
| `src/plugins/` | 2 | ~180 | `claude-plugins` | Built-in plugin registry |
| `src/query/` | 4 | ~650 | `claude-core` | Query config, deps, stop hooks, token budget |
| `src/remote/` | 4 | ~1,130 | `claude-bridge` | Remote session manager, WebSocket, SDK adapter |
| `src/schemas/` | 1 | ~220 | `claude-hooks` | Hook Zod schemas (command, prompt, agent, http) |
| `src/screens/` | 3 | ~6,000 | `claude-tui` | REPL (5K lines), ResumeConversation, Doctor |
| `src/server/` | 3 | ~360 | `claude-bridge` | Direct-connect session manager |
| `src/services/` | 127 | ~40,000 | Multiple crates | OAuth, API, analytics, compact, LSP, MCP, tools |
| `src/skills/` | 20 | ~8,000 | `claude-skills` | Skill loading, 17 bundled skills |
| `src/state/` | 6 | ~1,190 | `claude-state` | Store, AppState, selectors, onChange |
| `src/tasks/` | 12 | ~3,700 | `claude-tasks` | Shell, agent, remote, teammate, dream tasks |
| `src/tools/` | 100+ | ~30,000 | `claude-tools` | 40+ tools (bash, file, web, MCP, agent, etc.) |
| `src/types/` | 11 | ~3,000 | `claude-core` | Message, command, permission, plugin, hook types |
| `src/upstreamproxy/` | 2 | ~800 | `claude-net` | CCR upstream CONNECT-over-WebSocket proxy |
| `src/utils/` | 90+ | ~30,000 | `claude-utils` | Auth, config, git, path, shell, permissions |
| `src/vim/` | 5 | ~1,520 | `claude-vim` | Vim state machine (motions, operators, text objects) |
| `src/voice/` | 1 | ~80 | `claude-tui` | Voice mode feature gate |

**Total: ~1,900 files, ~512K lines of TypeScript**

---

## 4. Core Types & Traits

### Message Types

The message system uses discriminated unions extensively. In Rust, these become enums.

```rust
/// Root message type (discriminated union of all message kinds)
pub enum Message {
    User(UserMessage),
    Assistant(AssistantMessage),
    System(SystemMessage),
    Progress(ProgressMessage),
    Tombstone(TombstoneMessage),
}

pub struct UserMessage {
    pub uuid: Uuid,
    pub message: ApiUserMessage,
    pub timestamp: String,
    // tool results, pasted content, attachments...
}

pub struct AssistantMessage {
    pub uuid: Uuid,
    pub message: ApiAssistantMessage,  // content blocks: text, tool_use, thinking
    pub request_id: Option<String>,
    pub timestamp: String,
}

pub struct SystemMessage {
    pub uuid: Uuid,
    pub content: String,
    pub level: SystemLevel,  // info, warning, error
    pub subtype: Option<SystemSubtype>,
}
```

### Tool Trait

```rust
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn input_schema(&self) -> &JsonSchema;
    fn is_enabled(&self, ctx: &ToolUseContext) -> bool;
    fn is_read_only(&self) -> bool;
    fn needs_permissions(&self) -> bool;
    fn validate_input(&self, input: &Value) -> ValidationResult;

    async fn call(
        &self,
        input: Value,
        ctx: &mut ToolUseContext,
    ) -> Result<ToolResult, ToolError>;

    fn render_tool_use_message(&self, input: &Value) -> Option<String>;
    fn render_tool_result_message(&self, result: &ToolResult) -> Option<String>;
}
```

### Command Types

```rust
pub enum Command {
    Local(LocalCommand),
    LocalJsx(LocalJsxCommand),  // In Rust: renders via TUI
    Prompt(PromptCommand),
}

pub struct PromptCommand {
    pub name: String,
    pub description: String,
    pub allowed_tools: Option<Vec<String>>,
    pub content_length: usize,
    pub progress_message: String,
    pub get_prompt: Box<dyn Fn(&str, &ToolUseContext) -> Pin<Box<dyn Future<Output = Vec<ContentBlock>>>>>,
}
```

### Permission Types

```rust
pub enum PermissionMode {
    Default,
    Plan,
    BypassPermissions,
    Auto,
}

pub struct ToolPermissionContext {
    pub mode: PermissionMode,
    pub always_allow_rules: Vec<PermissionRule>,
    pub always_deny_rules: Vec<PermissionRule>,
    pub always_ask_rules: Vec<PermissionRule>,
    // ... per-source rules
}

pub enum PermissionResult {
    Approved,
    Denied { reason: String, retryable: bool },
}
```

---

## 5. Authentication & OAuth

### OAuth 2.0 with PKCE

The OAuth system supports three environments (production, staging, local) with identical flows:

```rust
pub struct OAuthConfig {
    pub base_api_url: String,
    pub console_authorize_url: String,
    pub claude_ai_authorize_url: String,
    pub token_url: String,
    pub client_id: String,
    pub mcp_proxy_url: String,
    // ... 12 more URLs
}

pub struct OAuthService {
    config: OAuthConfig,
    listener: AuthCodeListener,
}

impl OAuthService {
    /// Start browser-based OAuth flow with PKCE
    pub async fn start_oauth_flow(&self, scopes: &[&str]) -> Result<OAuthTokens, AuthError>;

    /// Manual paste-based auth code input (fallback)
    pub async fn handle_manual_auth_code(&self, code: &str) -> Result<OAuthTokens, AuthError>;
}
```

**PKCE flow** (must match exactly):
1. Generate 32-byte code verifier (base64url)
2. SHA256 hash → base64url code challenge
3. Open browser to authorize URL with challenge
4. Listen on ephemeral localhost port for redirect
5. Exchange code + verifier for tokens at token endpoint

**Token storage**: Platform-specific secure storage (macOS Keychain via `security` binary, Linux keyring, Windows Credential Manager).

**Auth priority chain**: OAuth tokens > API key helper > `ANTHROPIC_API_KEY` env > `~/.anthropic` file

### Multi-Provider Support

```rust
pub enum ApiProvider {
    FirstParty,       // api.anthropic.com
    Bedrock,          // AWS Bedrock (STS credentials)
    Vertex,           // Google Vertex AI (GoogleAuth)
    Foundry,          // Azure Foundry (DefaultAzureCredential)
}
```

Each provider has its own SDK client initialization, credential refresh, and header requirements.

---

## 6. State Management

### Global Session State (`bootstrap/state`)

A singleton with 212 fields and 200+ accessor functions. In Rust:

```rust
pub struct SessionState {
    // Identity
    pub session_id: SessionId,
    pub original_cwd: PathBuf,
    pub project_root: PathBuf,
    pub cwd: PathBuf,

    // API & Model
    pub main_loop_model_override: Option<ModelSetting>,
    pub model_usage: HashMap<String, ModelUsage>,

    // Cost tracking
    pub total_cost_usd: f64,
    pub total_api_duration: Duration,

    // Telemetry (OpenTelemetry)
    pub meter: Option<Meter>,
    pub session_counter: Option<Counter>,

    // ... 200 more fields
}

// Thread-safe global access
static STATE: Lazy<Mutex<SessionState>> = Lazy::new(|| Mutex::new(SessionState::default()));
```

### AppState (UI State)

```rust
pub struct AppState {
    // Settings
    pub settings: Settings,
    pub verbose: bool,
    pub main_loop_model: Option<ModelSetting>,

    // Permissions
    pub tool_permission_context: ToolPermissionContext,

    // Tasks
    pub tasks: HashMap<String, TaskState>,

    // Plugins & MCP
    pub plugins: PluginState,
    pub mcp: McpState,

    // Notifications
    pub notifications: NotificationState,

    // Bridge
    pub repl_bridge_enabled: bool,
    pub repl_bridge_connected: bool,

    // ... 150+ more fields
}

pub type AppStateStore = Store<AppState>;
```

### Store Pattern

```rust
pub struct Store<T> {
    state: Arc<RwLock<T>>,
    listeners: Arc<Mutex<Vec<Box<dyn Fn() + Send>>>>,
}

impl<T: Clone> Store<T> {
    pub fn get_state(&self) -> T;
    pub fn set_state(&self, updater: impl FnOnce(T) -> T);
    pub fn subscribe(&self, listener: impl Fn() + Send + 'static) -> impl FnOnce();
}
```

### onChange Side Effects

All side effects from state changes are centralized in one handler:

```rust
pub fn on_change_app_state(old: &AppState, new: &AppState) {
    // Permission mode sync → notify CCR/SDK
    // Model selection → persist to settings
    // Expanded view → persist to global config
    // Verbose flag → persist
    // Settings change → clear auth caches, re-apply env vars
}
```

---

## 7. Query Engine & Streaming

### Query Function

```rust
pub async fn query(params: QueryParams) -> impl Stream<Item = StreamEvent> {
    // 1. Build API request (system prompt, tools, messages)
    // 2. Call Anthropic SDK with streaming
    // 3. For each response chunk:
    //    - Parse tool_use blocks
    //    - Permission check
    //    - Execute tool (parallel for reads, serial for writes)
    //    - Feed result back
    // 4. Auto-compact if token budget exceeded
    // 5. Loop until stop_reason != tool_use
}
```

### Streaming Architecture

```rust
pub enum StreamEvent {
    RequestStart { request_id: String },
    TextDelta { text: String },
    ThinkingDelta { thinking: String },
    ToolUseStart { id: String, name: String },
    ToolUseInput { partial_json: String },
    ToolResult { id: String, result: ToolResult },
    Message(Message),
    Error(ApiError),
    Done { stop_reason: StopReason, usage: Usage },
}
```

### Context Compaction

When token usage approaches budget:

```rust
pub async fn auto_compact_if_needed(
    messages: &mut Vec<Message>,
    token_count: usize,
    budget: usize,
) -> Option<CompactResult> {
    // Strip images, group by API round
    // Call Claude to summarize older context
    // Replace with compact boundary message
    // Re-inject up to 5 recently-read files
}
```

### Token Budget Tracking

```rust
pub struct BudgetTracker {
    pub continuation_count: u32,
    pub last_delta_tokens: usize,
    pub last_global_turn_tokens: usize,
    pub started_at: Instant,
}

pub enum TokenBudgetDecision {
    Continue { nudge_message: String, pct: f64 },
    Stop { completion_event: Option<CompletionEvent> },
}
```

---

## 8. Tool System

### Tool Registry

40+ tools organized by category:

| Category | Tools |
|---|---|
| **File Operations** | FileRead, FileEdit, FileWrite, Glob, Grep, NotebookEdit |
| **Shell** | Bash (with security analysis, sandbox, permission checks) |
| **Web** | WebSearch, WebFetch |
| **Tasks** | TaskCreate, TaskGet, TaskUpdate, TaskList, TaskStop, TaskOutput, TodoWrite |
| **Agent** | Agent (spawn sub-agents), SendMessage (inter-agent) |
| **Planning** | EnterPlanMode, ExitPlanMode |
| **Worktree** | EnterWorktree, ExitWorktree |
| **Scheduling** | CronCreate, CronDelete, CronList, RemoteTrigger |
| **MCP** | MCPTool, ListMcpResources, ReadMcpResource, ToolSearch |
| **Skills** | Skill (invoke user-defined skills) |
| **User** | AskUserQuestion |
| **LSP** | LSP (goToDefinition, findReferences, hover, symbols) |
| **Config** | Config (get/set settings) |

### BashTool Security

The bash tool has the most complex permission system:

```rust
pub struct BashSecurityAnalysis {
    pub is_read_only: bool,
    pub is_search: bool,
    pub has_destructive_ops: bool,
    pub commands: Vec<ParsedCommand>,
    pub risk_level: RiskLevel,
}

/// Analyze command before execution
pub fn analyze_bash_command(command: &str) -> BashSecurityAnalysis;

/// Check if command matches permission rules
pub fn check_bash_permissions(
    command: &str,
    analysis: &BashSecurityAnalysis,
    context: &ToolPermissionContext,
) -> PermissionResult;
```

### Tool Execution Pipeline

```rust
pub async fn run_tool_use(
    tool: &dyn Tool,
    input: Value,
    ctx: &mut ToolUseContext,
) -> ToolExecutionResult {
    // 1. Validate input against schema
    // 2. Run pre-tool-use hooks
    // 3. Check permissions (auto/ask/deny)
    // 4. Execute tool
    // 5. Run post-tool-use hooks
    // 6. Truncate result if too large
    // 7. Store large results to disk with preview
}
```

---

## 9. Terminal UI

### Architecture

The TypeScript version uses React + Ink (a React renderer for terminals). The Rust rewrite uses **ratatui** with a custom layout engine.

```rust
// Main TUI application
pub struct App {
    state: AppStateStore,
    messages: Vec<Message>,
    input: TextInput,
    screen: Screen,      // Prompt | Transcript
    spinner: Spinner,
    permission_queue: VecDeque<PermissionRequest>,
}

pub enum Screen {
    Prompt,
    Transcript { search_query: Option<String> },
}
```

### Component Mapping (React/Ink → Ratatui)

| React Component | Rust Equivalent |
|---|---|
| `<Box>` | `ratatui::layout::Layout` |
| `<Text>` | `ratatui::text::Span` / `Line` |
| `useInput()` | crossterm event handler |
| `useEffect()` | Tokio task / channel |
| `useState()` | Struct field + signal |
| `useSyncExternalStore()` | Store subscription |
| ScrollBox | Virtual scroll widget |
| PromptInput | Custom multiline input widget |
| Messages | Message list widget with virtual scrolling |
| PermissionRequest | Modal dialog widget |
| Spinner | Animated spinner widget |

### Yoga Layout Port

The TypeScript codebase includes a pure-TS port of Yoga (flexbox layout). For Rust, use the `taffy` crate (Rust-native flexbox) which is API-compatible with Yoga concepts.

### Color Diff Rendering

```rust
pub struct ColorDiff {
    hunk: Hunk,
    file_path: String,
    theme: SyntaxTheme,
}

impl ColorDiff {
    /// Render diff with syntax highlighting and word-level changes
    pub fn render(&self, width: usize) -> Vec<StyledLine>;
}
```

Uses `syntect` for syntax highlighting (replaces highlight.js) and custom word-diff algorithm.

### File Index (Fuzzy Search)

```rust
pub struct FileIndex {
    paths: Vec<String>,
    lower_paths: Vec<String>,
    char_bits: Vec<u32>,    // a-z bitmap per path
    path_lens: Vec<u16>,
}

impl FileIndex {
    pub fn load_from_file_list(&mut self, files: Vec<String>);
    pub async fn load_from_file_list_async(&mut self, files: Vec<String>);
    pub fn search(&self, query: &str, limit: usize) -> Vec<SearchResult>;
}
```

Scoring matches nucleo/fzf-v2: boundary bonuses, camelCase bonuses, gap penalties, test file penalty.

---

## 10. Bridge & Remote Sessions

### Remote Control Architecture

31 files implementing the bridge system for controlling Claude Code from claude.ai/code:

```rust
pub struct BridgeApiClient {
    base_url: String,
    get_access_token: Box<dyn Fn() -> String>,
}

impl BridgeApiClient {
    pub async fn register_environment(&self, config: &BridgeConfig) -> (String, String);
    pub async fn poll_for_work(&self, env_id: &str) -> Option<WorkResponse>;
    pub async fn acknowledge_work(&self, env_id: &str, work_id: &str);
    pub async fn heartbeat_work(&self, env_id: &str, work_id: &str) -> HeartbeatResponse;
    // ... 5 more methods
}
```

### Transport Layer

```rust
pub enum Transport {
    WebSocket(WebSocketTransport),     // WS reads + writes
    Hybrid(HybridTransport),          // WS reads + HTTP POST writes
    Sse(SseTransport),                // SSE reads + HTTP POST writes (CCR v2)
}

pub trait ReplBridgeTransport {
    async fn write(&self, message: StdoutMessage) -> Result<()>;
    async fn write_batch(&self, messages: Vec<StdoutMessage>) -> Result<()>;
    fn is_connected(&self) -> bool;
    fn set_on_data(&self, callback: Box<dyn Fn(&str)>);
    fn set_on_close(&self, callback: Box<dyn Fn(Option<u16>)>);
}
```

### WebSocket Reconnection

```rust
pub struct WebSocketTransport {
    url: Url,
    state: WebSocketState,  // Idle | Connecting | Connected | Reconnecting | Closed
    reconnect_attempts: u32,
    // Exponential backoff: base 1s, max 30s, ±25% jitter
    // 10-minute reconnect budget
    // Sleep detection: gap > 60s resets budget
}
```

### JWT & Token Refresh

```rust
pub struct TokenRefreshScheduler {
    refresh_buffer_ms: u64,  // Default 5 min before expiry
    generation: AtomicU64,   // Invalidates stale refresh calls
}

impl TokenRefreshScheduler {
    pub fn schedule(&self, token: &str);
    pub fn schedule_from_expires_in(&self, expires_in_secs: u64);
    pub fn cancel(&self);
}
```

---

## 11. MCP Integration

### MCP Client

```rust
pub struct McpClient {
    transport: McpTransport,
    server_name: String,
    capabilities: ServerCapabilities,
}

pub enum McpTransport {
    Stdio { command: String, args: Vec<String> },
    Sse { url: String },
    Http { url: String },
    WebSocket { url: String },
}

impl McpClient {
    pub async fn list_tools(&self) -> Vec<McpToolDefinition>;
    pub async fn call_tool(&self, name: &str, args: Value) -> McpToolResult;
    pub async fn list_resources(&self) -> Vec<McpResource>;
    pub async fn read_resource(&self, uri: &str) -> McpResourceContent;
}
```

### MCP Tool Bridging

MCP tools are wrapped as native tools:

```rust
pub struct McpToolWrapper {
    client: Arc<McpClient>,
    tool_def: McpToolDefinition,
}

impl Tool for McpToolWrapper {
    fn name(&self) -> &str { &self.tool_def.name }
    fn input_schema(&self) -> &JsonSchema { &self.tool_def.input_schema }

    async fn call(&self, input: Value, _ctx: &mut ToolUseContext) -> Result<ToolResult> {
        let result = self.client.call_tool(&self.tool_def.name, input).await?;
        Ok(ToolResult::from_mcp(result))
    }
}
```

---

## 12. Plugin & Skill Systems

### Plugin Architecture

```rust
pub struct LoadedPlugin {
    pub name: String,
    pub manifest: PluginManifest,
    pub path: PathBuf,
    pub source: String,          // "{name}@{marketplace}"
    pub enabled: bool,
    pub commands: Vec<Command>,
    pub agents: Vec<AgentDefinition>,
    pub hooks: Option<HooksSettings>,
    pub mcp_servers: HashMap<String, McpServerConfig>,
}

pub struct PluginLoader {
    pub fn load_all_plugins(&self) -> PluginLoadResult;
    pub fn install_plugin(&self, spec: &str, scope: Scope) -> Result<()>;
    pub fn uninstall_plugin(&self, name: &str) -> Result<()>;
}
```

### Skill System

Skills are markdown files with YAML frontmatter:

```rust
pub struct SkillDefinition {
    pub name: String,
    pub description: String,
    pub allowed_tools: Option<Vec<String>>,
    pub when_to_use: Option<String>,
    pub model: Option<String>,
    pub hooks: Option<HooksSettings>,
    pub context: SkillContext,  // Inline | Fork
    pub paths: Option<Vec<String>>,  // Conditional activation patterns
    pub get_prompt: Box<dyn Fn(&str, &ToolUseContext) -> Pin<Box<dyn Future<Output = Vec<ContentBlock>>>>>,
}

/// Load skills from all sources
pub async fn get_skill_dir_commands(cwd: &Path) -> Vec<Command> {
    // 1. Managed skills (policy-level)
    // 2. User skills (~/.claude/skills)
    // 3. Project skills (.claude/skills + parents)
    // 4. Additional dirs (--add-dir)
    // 5. Legacy commands (/commands/)
    // Deduplicate by file identity (realpath)
    // Separate conditional skills (with paths) for lazy activation
}
```

### Bundled Skills (17 total)

| Skill | Purpose |
|---|---|
| `update-config` | Modify settings.json (permissions, hooks, env) |
| `keybindings-help` | Keyboard shortcut customization guide |
| `verify` | Code verification wrapper |
| `debug` | Enable debug logging + read log tail |
| `simplify` | 3 parallel agents reviewing code quality |
| `batch` | Research + parallel worktree execution |
| `claude-api` | Claude API documentation (247KB bundled) |
| `claude-in-chrome` | Browser automation via MCP |
| `loop` | Recurring prompt scheduling |
| `schedule` | Remote trigger management (CCR) |
| `skillify` | Generate skill from session (ANT-only) |
| `remember` | Memory review + promotion (ANT-only) |
| `stuck` | Process diagnostics (ANT-only) |
| `lorem-ipsum` | Token count testing (ANT-only) |
| `dream` | Memory consolidation |
| `hunter` | Bug finding |
| `runSkillGenerator` | Skill auto-generation |

---

## 13. Configuration & Settings

### Settings Hierarchy

```
Priority (highest to lowest):
1. CLI flags (--effort, --model, etc.)
2. Environment variables (CLAUDE_CODE_*)
3. Remote Managed Settings (org policy)
4. Project config (.claude/settings.json)
5. Local settings (~/.claude/settings.local.json)
6. User settings (~/.claude/settings.json)
```

### Configuration Files

```rust
/// Global config: ~/.claude.json
pub struct GlobalConfig {
    pub user_id: String,
    pub has_completed_onboarding: bool,
    pub trust_dialog_accepted_dirs: HashMap<String, bool>,
    pub migration_version: u32,
    pub num_startups: u32,
    // ... 30+ fields
}

/// Project config: .claude.json (per-directory)
pub struct ProjectConfig {
    pub allowed_tools: Vec<String>,
    pub mcp_servers: HashMap<String, McpServerConfig>,
    pub last_session_metrics: Option<SessionMetrics>,
    // ...
}

/// Settings: ~/.claude/settings.json
pub struct Settings {
    pub model: Option<String>,
    pub effort: Option<EffortLevel>,
    pub permissions: Option<PermissionSettings>,
    pub hooks: Option<HooksSettings>,
    pub env: Option<HashMap<String, String>>,
    pub enabled_plugins: Option<HashMap<String, bool>>,
    pub spinner_verbs: Option<SpinnerVerbsConfig>,
    // ... 50+ fields (Zod schema → serde)
}
```

### OAuth Configuration

Three environments with identical structure:

```rust
pub fn get_oauth_config() -> OAuthConfig {
    match get_oauth_config_type() {
        OAuthEnv::Production => OAuthConfig {
            base_api_url: "https://platform.claude.com".into(),
            client_id: "9d1c250a-e61b-44d9-88ed-5944d1962f5e".into(),
            mcp_proxy_url: "https://mcp-proxy.anthropic.com".into(),
            // ...
        },
        OAuthEnv::Staging => { /* staging.ant.dev endpoints */ },
        OAuthEnv::Local => { /* localhost endpoints */ },
    }
}
```

### Migrations

11 migration functions run sequentially on version mismatch:

```rust
pub const CURRENT_MIGRATION_VERSION: u32 = 11;

pub fn run_migrations() {
    migrate_auto_updates_to_settings();
    migrate_bypass_permissions_accepted_to_settings();
    migrate_enable_all_project_mcp_servers_to_settings();
    reset_pro_to_opus_default();
    migrate_sonnet_1m_to_sonnet_45();
    migrate_legacy_opus_to_current();
    migrate_sonnet_45_to_sonnet_46();
    migrate_opus_to_opus_1m();
    migrate_repl_bridge_enabled_to_remote_control_at_startup();
    reset_auto_mode_opt_in_for_default_offer();
    migrate_fennec_to_opus();
}
```

---

## 14. Memory System

### Auto-Memory Architecture

```rust
/// Memory directory: ~/.claude/projects/{sanitized-git-root}/memory/
pub struct MemorySystem {
    pub memory_dir: PathBuf,
    pub team_dir: Option<PathBuf>,  // {memory_dir}/team/ (if TEAMMEM enabled)
}

/// Four memory types (closed taxonomy)
pub enum MemoryType {
    User,       // Role, goals, preferences
    Feedback,   // Guidance on approach (what to avoid/repeat)
    Project,    // Ongoing work, initiatives, deadlines
    Reference,  // Pointers to external systems
}

/// Memory file: frontmatter + content
pub struct MemoryFile {
    pub name: String,
    pub description: String,
    pub memory_type: MemoryType,
    pub content: String,
    pub mtime: SystemTime,
}
```

### Memory Scanning & Selection

```rust
/// Scan memory directory for .md files with frontmatter
pub async fn scan_memory_files(dir: &Path) -> Vec<MemoryHeader>;

/// Use Sonnet to select relevant memories for a query
pub async fn find_relevant_memories(
    query: &str,
    memory_dir: &Path,
    already_surfaced: &HashSet<String>,
) -> Vec<RelevantMemory>;
```

### Team Memory Security

```rust
/// Two-pass path validation for team memory writes
pub async fn validate_team_mem_write_path(path: &Path) -> Result<PathBuf, PathTraversalError> {
    // Pass 1: resolve() containment check
    // Pass 2: realpath() on deepest existing ancestor + symlink check
    // Rejects: null bytes, URL-encoded traversals, Unicode normalization attacks
}
```

---

## 15. Task & Agent Orchestration

### Task Types

```rust
pub enum TaskState {
    LocalShell(LocalShellTaskState),
    LocalAgent(LocalAgentTaskState),
    RemoteAgent(RemoteAgentTaskState),
    InProcessTeammate(InProcessTeammateTaskState),
    Dream(DreamTaskState),
    LocalWorkflow(LocalWorkflowTaskState),
    MonitorMcp(MonitorMcpTaskState),
}

pub enum TaskStatus {
    Pending,
    Running,
    Completed,
    Failed,
    Killed,
}
```

### Agent Spawning

```rust
pub async fn spawn_agent(
    prompt: &str,
    agent_type: &str,
    tools: Vec<Arc<dyn Tool>>,
    parent_abort: Option<&AbortController>,
) -> AgentHandle {
    // 1. Register task in AppState
    // 2. Create child abort controller
    // 3. Build agent-specific system prompt
    // 4. Run query() in isolated context
    // 5. Track progress (tokens, tool uses, activities)
    // 6. On completion: enqueue notification
}
```

### Coordinator Mode

```rust
/// Multi-agent orchestration mode
pub fn get_coordinator_system_prompt() -> String {
    // 370-line prompt covering:
    // - Role (coordinator, not direct executor)
    // - Tools (Agent, SendMessage, TaskStop)
    // - Worker capabilities
    // - Task workflow phases (research → synthesis → implementation → verification)
    // - Writing worker prompts (self-contained, specific)
    // - Example session
}
```

---

## 16. CLI Entry Points & Commands

### Main Entry Point

```rust
fn main() {
    // 1. Profile checkpoints
    // 2. Parse CLI args (clap)
    // 3. Run migrations
    // 4. Initialize: bootstrap state, permissions, models, settings
    // 5. Setup: hooks, worktrees, background jobs
    // 6. Route to: headless query | interactive REPL | bridge mode
}
```

### Command System

80+ commands across three types:

```rust
pub enum CommandType {
    /// Simple sync execution, returns text
    Local,
    /// Interactive TUI rendering
    LocalJsx,
    /// Generates prompt for Claude execution
    Prompt,
}
```

Key commands to preserve exactly:
- `/login`, `/logout` — OAuth flow
- `/commit`, `/commit-push-pr` — Git workflow with attribution
- `/init` — Multi-phase CLAUDE.md generation
- `/resume`, `/continue` — Session restoration
- `/compact` — Manual context compaction
- `/model`, `/effort`, `/fast` — Model configuration
- `/mcp` — MCP server management
- `/plugin` — Plugin marketplace

---

## 17. Networking & Proxy

### Proxy Configuration

```rust
pub struct ProxyConfig {
    pub proxy_url: Option<String>,
    pub no_proxy: Vec<NoProxyPattern>,
    pub mtls_cert: Option<PathBuf>,
    pub mtls_key: Option<PathBuf>,
    pub ca_bundle: Option<PathBuf>,
}

pub fn should_bypass_proxy(host: &str) -> bool {
    // Parse NO_PROXY: exact hostname, domain suffix with '.', wildcard '*', port-specific
}

pub fn create_http_client(config: &ProxyConfig) -> reqwest::Client;
pub fn create_ws_connector(config: &ProxyConfig) -> tokio_tungstenite::Connector;
```

### Upstream Proxy (CCR)

CONNECT-over-WebSocket relay for credential injection in container environments:

```rust
pub struct UpstreamProxyRelay {
    pub port: u16,
    pub stop: Box<dyn FnOnce()>,
}

/// Hand-encoded protobuf for tunnel chunks
pub fn encode_chunk(data: &[u8]) -> Vec<u8>;
pub fn decode_chunk(buf: &[u8]) -> Option<Vec<u8>>;

/// Start local TCP relay that tunnels CONNECT to WebSocket
pub async fn start_relay(ws_url: &str, session_id: &str, token: &str) -> UpstreamProxyRelay;
```

---

## 18. Native Modules

### Color Diff (syntax highlighting + word diff)

Replace highlight.js with `syntect`, replace `diff` npm package with `similar` crate:

```rust
pub struct ColorDiff {
    hunk: Hunk,
    theme: Theme,
    language: Option<String>,
}

impl ColorDiff {
    pub fn render(&self, width: usize) -> Vec<StyledLine> {
        // 1. Detect language from file path/shebang
        // 2. Highlight lines via syntect
        // 3. Word-diff adjacent +/- pairs
        // 4. Apply background colors for changes
        // 5. Wrap to console width (grapheme-aware)
        // 6. Add line numbers and markers
    }
}
```

### File Index (fuzzy search)

Port the nucleo-equivalent scoring algorithm:

```rust
const SCORE_MATCH: i32 = 16;
const BONUS_BOUNDARY: i32 = 8;
const BONUS_CAMEL: i32 = 6;
const BONUS_CONSECUTIVE: i32 = 4;
const BONUS_FIRST_CHAR: i32 = 8;
const PENALTY_GAP_START: i32 = 3;
const PENALTY_GAP_EXTENSION: i32 = 1;
```

### Yoga Layout

Use `taffy` crate (pure Rust flexbox implementation) instead of porting the 2600-line Yoga implementation.

---

## 19. Keybindings

### System Architecture

```rust
pub struct KeybindingSystem {
    bindings: Vec<ParsedBinding>,
    active_contexts: HashSet<KeybindingContext>,
    pending_chord: Option<Vec<ParsedKeystroke>>,
    handler_registry: HashMap<String, Vec<HandlerRegistration>>,
}

pub struct ParsedKeystroke {
    pub key: String,
    pub ctrl: bool,
    pub alt: bool,
    pub shift: bool,
    pub meta: bool,
    pub super_key: bool,
}

pub type Chord = Vec<ParsedKeystroke>;

pub enum ChordResolveResult {
    Match { action: String },
    ChordStarted { pending: Vec<ParsedKeystroke> },
    ChordCancelled,
    Unbound,
    None,
}
```

17 contexts: Global, Chat, Autocomplete, Confirmation, Help, Transcript, HistorySearch, Task, ThemePicker, Settings, Tabs, Attachments, Footer, MessageSelector, DiffDialog, ModelPicker, Select.

60+ actions with platform-specific defaults (macOS opt vs Linux/Windows alt).

---

## 20. Vim Mode

### State Machine

```rust
pub enum VimMode {
    Insert { inserted_text: String },
    Normal { command: CommandState },
}

pub enum CommandState {
    Idle,
    Count { digits: String },
    Operator { op: Operator, count: u32 },
    OperatorCount { op: Operator, count: u32, digits: String },
    OperatorFind { op: Operator, count: u32, find: FindType },
    OperatorTextObj { op: Operator, count: u32, scope: TextObjScope },
    Find { find: FindType, count: u32 },
    G { count: u32 },
    OperatorG { op: Operator, count: u32 },
    Replace { count: u32 },
    Indent { dir: IndentDir, count: u32 },
}

pub fn transition(
    state: &CommandState,
    input: &str,
    ctx: &mut TransitionContext,
) -> TransitionResult;
```

### Supported Operations

- **Motions**: h, l, j, k, w, b, e, W, B, E, 0, ^, $, G, gj, gk, gg
- **Operators**: d (delete), c (change), y (yank) — with motions, finds, text objects
- **Text Objects**: iw, aw, iW, aW, i", a", i', a', i`, a`, i(, a(, i[, a[, i{, a{, i<, a<
- **Find**: f, F, t, T — with ; and , repeat
- **Commands**: x, r, ~, J, p, P, D, C, Y, o, O, >>, <<, u, .

---

## 21. Analytics & Telemetry

### Event Logging

```rust
pub trait AnalyticsSink: Send + Sync {
    fn log_event(&self, name: &str, metadata: HashMap<String, Value>);
}

pub fn log_event(name: &str, metadata: HashMap<String, Value>) {
    // Queue events before sink attachment
    // Dispatch to all attached sinks
    // Sinks: Datadog, 1P event logger, OpenTelemetry
}
```

### GrowthBook Feature Flags

```rust
/// Cached feature flag evaluation
pub fn get_feature_value_cached<T: DeserializeOwned>(
    flag: &str,
    default: T,
) -> T;

/// Blocking evaluation (waits for fetch)
pub async fn get_feature_value_blocking<T: DeserializeOwned>(
    flag: &str,
    default: T,
) -> T;
```

---

## 22. Migration & Compatibility

### File Format Compatibility

All existing configuration and storage files must remain readable:

| File | Format | Path |
|---|---|---|
| Global config | JSON | `~/.claude.json` |
| Settings | JSON | `~/.claude/settings.json` |
| Project config | JSON | `.claude.json` |
| Session transcripts | JSONL | `~/.claude/projects/{hash}/` |
| Memory files | Markdown + YAML frontmatter | `~/.claude/projects/{hash}/memory/` |
| Keybindings | JSON | `~/.claude/keybindings.json` |
| MCP config | JSON | `.mcp.json` |
| Skills | Markdown + YAML frontmatter | `.claude/skills/{name}/SKILL.md` |
| Hooks | JSON (in settings) | `.claude/settings.json` |
| Task lists | JSON | `~/.claude/tasks/{id}/` |

### Session Resume Compatibility

Existing TypeScript session transcripts must be loadable by the Rust version for `--resume` / `--continue`.

---

## 23. Rust Crate Dependencies

| Purpose | TypeScript | Rust Crate |
|---|---|---|
| HTTP client | axios | `reqwest` |
| WebSocket | ws / global WebSocket | `tokio-tungstenite` |
| JSON | JSON.parse/stringify | `serde_json` |
| Schema validation | zod | `serde` + `validator` |
| CLI parsing | commander.js | `clap` |
| Terminal UI | React + Ink | `ratatui` + `crossterm` |
| Flexbox layout | Yoga (custom port) | `taffy` |
| Syntax highlighting | highlight.js | `syntect` |
| Diff | diff (npm) | `similar` |
| OAuth | custom | `oauth2` |
| UUID | crypto.randomUUID | `uuid` |
| Regex | RegExp | `regex` |
| Async runtime | Bun event loop | `tokio` |
| Channels | N/A | `tokio::sync::mpsc` |
| File watching | chokidar | `notify` |
| Glob patterns | ripgrep (external) | `globset` + `ignore` |
| Content search | ripgrep (external) | `grep` (ripgrep library) |
| Git operations | git CLI | `git2` or git CLI |
| Shell execution | child_process | `tokio::process::Command` |
| Secure storage | macOS security binary | `keyring` |
| mTLS | Node TLS | `rustls` + `native-tls` |
| Proxy | https-proxy-agent | `reqwest` proxy support |
| Markdown parsing | custom | `pulldown-cmark` |
| YAML parsing | yaml (npm) | `serde_yaml` |
| Grapheme handling | Intl.Segmenter | `unicode-segmentation` |
| Semver | semver (npm) | `semver` |
| Protobuf (hand-encoded) | custom | `prost` or hand-encode |
| SSE | fetch + ReadableStream | `eventsource-client` or `reqwest` |
| Cron parsing | custom | `cron` |
| Fuzzy matching | Fuse.js + custom | `nucleo` or custom |
| QR code | qrcode (npm) | `qrcode` |
| Process management | child_process | `nix` (Unix signals) |
| File locking | proper-lockfile | `fs2` |
| Base64 | Buffer | `base64` |
| SHA256 | crypto | `sha2` |
| String width | custom stringWidth | `unicode-width` |
| Color output | chalk | `colored` or ANSI direct |
| OpenTelemetry | @opentelemetry/* | `opentelemetry` + `opentelemetry-otlp` |

---

## 24. Implementation Phases

### Phase 1: Foundation (Weeks 1-4)
- `claude-utils`: File operations, path handling, git utilities, shell execution
- `claude-config`: Settings loading, global config, project config, migrations
- `claude-auth`: OAuth PKCE flow, API key management, secure storage
- `claude-state`: Store pattern, SessionState, basic AppState

### Phase 2: Core Engine (Weeks 5-8)
- `claude-core`: Message types, query engine, streaming, tool loop
- `claude-tools`: File tools (Read, Edit, Write, Glob, Grep), BashTool
- `claude-net`: Proxy, mTLS, HTTP client creation
- `claude-analytics`: Event logging, GrowthBook client

### Phase 3: Tool Ecosystem (Weeks 9-12)
- `claude-tools`: Remaining tools (Web, Task, Agent, LSP, Config, Worktree)
- `claude-mcp`: MCP client, all transports, tool bridging
- `claude-hooks`: Hook execution (command, prompt, agent, http)
- `claude-memory`: Memory system, scanning, team memory, selection

### Phase 4: UI & Interaction (Weeks 13-16)
- `claude-tui`: Ratatui components, message rendering, input handling
- `claude-vim`: Vim state machine
- `claude-native`: Color diff, file index
- `claude-cli`: REPL screen, commands, keybindings

### Phase 5: Advanced Features (Weeks 17-20)
- `claude-bridge`: Remote control, CCR v1/v2, WebSocket transports
- `claude-tasks`: All task types (shell, agent, remote, teammate, dream)
- `claude-plugins`: Plugin loader, marketplace, builtin plugins
- `claude-skills`: Skill loading, all 17 bundled skills

### Phase 6: Integration & Polish (Weeks 21-24)
- Full integration testing against TypeScript version
- Session resume compatibility
- Performance benchmarking
- Edge case handling (Unicode, large files, network errors)
- Documentation

---

## Appendix: System Prompt Preservation

The system prompt (`src/constants/prompts.ts`, 54K bytes) must be reproduced exactly. Key sections:

1. **CLI System Prompt Prefix** — "You are Claude Code, Anthropic's official CLI for Claude."
2. **System Rules** — Permission modes, tool usage guidelines, output efficiency
3. **Environment Info** — Platform, shell, OS, model, knowledge cutoff
4. **Tool-Specific Guidance** — Per-tool instructions based on available tools
5. **MCP Instructions** — Generated from connected MCP servers
6. **Output Style** — User-customizable response formatting
7. **Hooks Documentation** — Current hook configuration for model awareness
8. **Dynamic Boundary** — Separates cacheable from session-specific content

The prompt caching boundary (`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`) must remain in the same position to preserve prompt cache hit rates.
