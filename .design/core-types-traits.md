# Feature: Core Types & Traits (`claude-types` crate)

## Summary
Define the shared type system for the Rust rewrite as the `claude-types` crate — the foundational dependency for all other crates. This includes message types (discriminated union of User/Assistant/System/Progress/Tombstone), the `Tool` trait (generic over Input/Output/Progress with ~50 methods), command types (Local/LocalJsx/Prompt with 30+ base fields), permission types (7 modes, 3 behaviors, 7 rule sources), branded ID types (SessionId, AgentId), hook types, plugin types, and session log types. All types must serialize/deserialize identically with the TypeScript version for cross-version compatibility.

## Requirements

- REQ-1: Message types must be a `#[non_exhaustive]` enum with variants User, Assistant, System, Progress, Tombstone — matching the TypeScript discriminated union in `src/types/logs.ts`. Each variant carries a typed struct with all fields from the TypeScript source (UserMessage: uuid, message, timestamp, cwd, userType, entrypoint, sessionId, version, gitBranch, slug; AssistantMessage: uuid, message, request_id, timestamp; SystemMessage: uuid, content, level, subtype)
- REQ-2: The `Tool` trait must be generic over `Input: DeserializeOwned`, `Output: Serialize`, and `Progress: Serialize`, with all 47 members from `src/Tool.ts`. Core execution: `call()`, `description()`, `prompt()`, `input_schema`, `input_json_schema`, `output_schema`, `max_result_size_chars`. Capability checks: `is_enabled()`, `is_read_only()`, `is_destructive()`, `is_concurrency_safe()`, `is_open_world()`, `is_search_or_read_command()`, `requires_user_interaction()`, `interrupt_behavior()`. Permissions/validation: `check_permissions()`, `validate_input()`, `prepare_permission_matcher()`, `get_path()`. Identity: `name`, `aliases`, `user_facing_name()`, `user_facing_name_background_color()`, `search_hint`. MCP/deferred: `is_mcp`, `mcp_info`, `is_lsp`, `should_defer`, `always_load`, `strict`, `is_transparent_wrapper()`. Classification: `to_auto_classifier_input()`, `inputs_equivalent()`, `backfill_observable_input()`. Summary: `get_tool_use_summary()`, `get_activity_description()`. Result handling: `map_tool_result_to_tool_result_block_param()`, `extract_search_text()`, `is_result_truncated()`. Rendering (8 methods, delegated to separate `ToolRenderer` trait or kept on Tool): `render_tool_result_message()`, `render_tool_use_message()`, `render_tool_use_tag()`, `render_tool_use_progress_message()`, `render_tool_use_queued_message()`, `render_tool_use_rejected_message()`, `render_tool_use_error_message()`, `render_grouped_tool_use()`. Methods with default implementations in TypeScript's `buildTool()` must provide Rust default trait method implementations
- REQ-3: Command types must model the three-variant union: `Local` (sync execution returning text/compact/skip), `LocalJsx` (TUI rendering via callback), `Prompt` (generates content blocks for model execution). The `CommandBase` struct must include all 18 fields from `src/types/command.ts`: name, aliases, description, availability, is_enabled, is_hidden, argument_hint, when_to_use, version, disable_model_invocation, user_invocable, loaded_from, kind, immediate, is_sensitive, user_facing_name, is_mcp, has_user_specified_description
- REQ-4: Permission types must model: `PermissionMode` enum with 7 variants (AcceptEdits, BypassPermissions, Default, DontAsk, Plan, Auto, Bubble), `PermissionBehavior` enum with 3 variants (Allow, Deny, Ask), `PermissionRuleSource` enum with 8 variants (UserSettings, ProjectSettings, LocalSettings, FlagSettings, PolicySettings, CliArg, Command, Session), and `PermissionResult` as a 4-variant enum (Allow, Ask, Deny, Passthrough) each carrying decision metadata including reason discriminants
- REQ-5: Branded ID types `SessionId` and `AgentId` must use Rust newtype wrappers over `String` with validation: AgentId must match pattern `^a(?:.+-)?[0-9a-f]{16}$`. Both must implement `Serialize`, `Deserialize`, `Clone`, `Hash`, `Eq`, `Display`
- REQ-6: Hook types must model `HookEvent` variants, `HookResult` with outcome enum (Success, Blocking, NonBlockingError, Cancelled), and `AggregatedHookResult` — matching the Zod schemas in `src/schemas/hooks.ts` and types in `src/types/hooks.ts`
- REQ-7: Plugin types must model `LoadedPlugin` with all fields: name, manifest, path, source, repository, enabled, is_builtin, sha, commands_path, commands_paths, commands_metadata, agents_path, agents_paths, skills_path, skills_paths, output_styles_path, output_styles_paths, hooks_config, mcp_servers, lsp_servers, settings. `PluginError` must be a 25-variant enum matching the TypeScript discriminated union (path-not-found, git-auth-failed, manifest-parse-error, plugin-not-found, mcp-config-invalid, lsp-server-crashed, etc.)
- REQ-8: All types must derive `serde::Serialize` and `serde::Deserialize` with `#[serde(rename_all = "camelCase")]` to match TypeScript's camelCase JSON keys. Optional fields must use `#[serde(skip_serializing_if = "Option::is_none")]` to match TypeScript's undefined-omission behavior
- REQ-9: Session log entry types must model all 20 variants of the `Entry` union from `src/types/logs.ts`: TranscriptMessage, SummaryMessage, CustomTitleMessage, AiTitleMessage, LastPromptMessage, TaskSummaryMessage, TagMessage, AgentNameMessage, AgentColorMessage, AgentSettingMessage, PRLinkMessage, FileHistorySnapshotMessage, AttributionSnapshotMessage, ContentReplacementEntry, QueueOperationMessage, SpeculationAcceptMessage, ModeEntry, WorktreeStateEntry, ContextCollapseCommitEntry, ContextCollapseSnapshotEntry
- REQ-10: The `ToolResult` type must support both text content and structured content blocks, with truncation tracking (`is_result_truncated`) and disk persistence for oversized results (controlled by `max_result_size_chars`)

## Acceptance Criteria

- [ ] AC-1: Roundtrip serde test — serialize each type to JSON and back, verifying field names match TypeScript's camelCase output (test against fixtures extracted from real TypeScript session transcripts)
- [ ] AC-2: Every variant of Message enum can be deserialized from a TypeScript-generated JSONL transcript line without error — verified by loading 10+ real session transcripts
- [ ] AC-3: The `Tool` trait compiles with a mock implementation that provides all required methods and uses default implementations for optional methods — verified by `cargo test`
- [ ] AC-4: SessionId and AgentId newtypes reject invalid strings at construction time — verified by property tests with invalid inputs
- [ ] AC-5: Permission types can express every combination present in the TypeScript test suite — verified by translating TypeScript permission tests to Rust
- [ ] AC-6: `PluginError` enum covers all 25 error variants from TypeScript — verified by a test that constructs each variant
- [ ] AC-7: The `CommandBase` struct can deserialize from JSON emitted by TypeScript's command registry — verified by fixture tests
- [ ] AC-8: No `unwrap()` or `panic!()` in any type's `Deserialize` implementation — all parse errors return `Result::Err`

## Architecture

The `claude-types` crate is a pure data crate with no business logic — only type definitions, serde implementations, and validation. It sits at the bottom of the dependency graph, depended on by every other crate.

**File structure:**
```
crates/claude-types/
├── src/
│   ├── lib.rs          (re-exports all modules)
│   ├── message.rs      (Message enum, UserMessage, AssistantMessage, SystemMessage, etc.)
│   ├── tool.rs         (Tool trait, ToolResult, ToolError, ToolUseContext, ValidationResult)
│   ├── command.rs      (Command enum, CommandBase, LocalCommand, PromptCommand, etc.)
│   ├── permission.rs   (PermissionMode, PermissionBehavior, PermissionRule, PermissionResult)
│   ├── ids.rs          (SessionId, AgentId newtypes with validation)
│   ├── hooks.rs        (HookEvent, HookResult, HookCallback, AggregatedHookResult)
│   ├── plugin.rs       (LoadedPlugin, PluginManifest, PluginError, PluginComponent)
│   ├── logs.rs         (Entry union, SerializedMessage, LogOption, PersistedWorktreeSession)
│   └── text_input.rs   (BaseTextInputProps, VimMode, QueuedCommand, QueuePriority)
```

**Key design decisions:**
- The `Tool` trait uses `async_trait` for the `call()` method since tools execute async I/O
- `ToolUseContext` is a struct passed by mutable reference, not a trait, matching TypeScript's concrete object pattern
- Permission types use the builder pattern for constructing `PermissionRule` chains
- All enums use `#[non_exhaustive]` to allow adding variants without breaking downstream crates
- The `SerializedMessage` type in `logs.rs` is the on-disk format (JSONL), distinct from the in-memory `Message` type — conversion between them is explicit via `From`/`Into` implementations

**serde strategy:**
- `#[serde(tag = "type")]` for internally-tagged enums (matching TypeScript discriminated unions)
- `#[serde(untagged)]` only where TypeScript uses runtime type checking without a discriminant field
- `#[serde(default)]` for fields that TypeScript initializes to `undefined` (maps to `Option<T>::None`)
- Custom deserializers for branded types that validate on construction

## Open Questions

### Q1: Tool trait rendering methods
**Decision**: Decouple rendering into a separate `ToolRenderer` trait in `claude-tui`. The `Tool` trait in `claude-types` remains UI-agnostic (no ratatui dependency), keeping it usable in headless/SDK contexts. The 8 rendering methods are defined on `ToolRenderer<T: Tool>` in `claude-tui`, which wraps each tool and maps its output to ratatui widgets. This is the clean Rust pattern — data types don't depend on UI frameworks.

### Q2: ToolUseContext lifetime management
**Decision**: Use `Arc<Mutex<...>>` (specifically `Arc<parking_lot::Mutex<...>>`) for shared references in `ToolUseContext`. This matches TypeScript's shared-reference pattern, keeps tool signatures simple (`&ToolUseContext` instead of `ToolUseContext<'a, 'b, 'c>`), and the runtime cost is negligible — tool calls involve I/O (disk, network, process spawning) that dwarfs mutex overhead. `ToolUseContext` will hold `Arc<RwLock<SessionState>>`, `Arc<RwLock<AppState>>`, `CancellationToken`, and `Option<Arc<McpClient>>`.

## Out of Scope

- Business logic (validation rules, permission checking algorithms) — those live in their respective crates
- Concrete tool implementations — those live in `claude-tools`
- TUI widget types — those live in `claude-tui`
- API request/response types for the Anthropic SDK — those live in `claude-core`
