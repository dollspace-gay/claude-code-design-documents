# Feature: State Management (`claude-state` crate)

## Summary
Implement the state management system for the Rust rewrite as the `claude-state` crate. This includes the SessionState singleton (98 top-level fields, ~210 exported functions covering identity, cost tracking, per-turn metrics, token tracking, model/API state, telemetry counters, logging/tracing, agent colors, configuration, auth tokens, plugins, permissions, scheduled tasks, teams, UI state, SDK integration, skills, performance monitoring, remote mode, prompt cache, and error logging), the AppState UI state tree with immutable update semantics, the generic `Store<T>` pattern with listener-based change notifications, and the `on_change_app_state` side-effect handler.

## Requirements

- REQ-1: SessionState must be a thread-safe singleton containing all 98 fields from `src/bootstrap/state.ts` `getInitialState()`, wrapped in `Arc<RwLock<SessionState>>`. Access must be through getter/setter functions matching the ~210 TypeScript exports. Fields grouped by category:
  - **Identity (7)**: session_id, parent_session_id, original_cwd, project_root, cwd, session_project_dir, additional_directories_for_claude_md
  - **Cost/Usage (7)**: total_cost_usd, total_api_duration, total_api_duration_without_retries, total_tool_duration, total_lines_added, total_lines_removed, has_unknown_model_cost
  - **Per-Turn Metrics (6+3)**: turn_hook_duration_ms, turn_tool_duration_ms, turn_classifier_duration_ms, turn_tool_count, turn_hook_count, turn_classifier_count (+ 3 module-scope vars translated to fields: output_tokens_at_turn_start, current_turn_token_budget, budget_continuation_count)
  - **Time (3)**: start_time, last_interaction_time, last_api_completion_timestamp
  - **Model/API (5)**: model_usage, main_loop_model_override, initial_main_loop_model, model_strings, sdk_betas
  - **API Request Cache (4)**: last_api_request, last_api_request_messages, last_main_request_id, last_classifier_requests
  - **Claude.md Cache (3)**: cached_claude_md_content, system_prompt_section_cache, last_emitted_date
  - **Telemetry (14)**: meter, session_counter, loc_counter, pr_counter, commit_counter, cost_counter, token_counter, code_edit_tool_decision_counter, active_time_counter, stats_store, logger_provider, event_logger, meter_provider, tracer_provider
  - **Agent Colors (2)**: agent_color_map, agent_color_index
  - **Configuration (11)**: is_interactive, kairos_active, strict_tool_result_pairing, sdk_agent_progress_summaries_enabled, user_msg_opt_in, client_type, session_source, question_preview_format, flag_settings_path, flag_settings_inline, allowed_setting_sources
  - **Auth Tokens (3)**: session_ingress_token, oauth_token_from_fd, api_key_from_fd
  - **Plugins (3)**: inline_plugins, chrome_flag_override, use_cowork_plugins
  - **Permissions (3)**: session_bypass_permissions_mode, session_trust_accepted, session_persistence_disabled
  - **Scheduled Tasks (2)**: scheduled_tasks_enabled, session_cron_tasks
  - **Teams (1)**: session_created_teams
  - **UI State (4)**: has_exited_plan_mode, needs_plan_mode_exit_attachment, needs_auto_mode_exit_attachment, lsp_recommendation_shown_this_session
  - **SDK (3)**: init_json_schema, registered_hooks, plan_slug_cache
  - **Teleport (1)**: teleported_session_info
  - **Skills (1)**: invoked_skills
  - **Performance (1)**: slow_operations (TTL 10s, max 10 items)
  - **Remote (3)**: is_remote_mode, main_thread_agent_type, direct_connect_server_url
  - **Channels (2)**: allowed_channels, has_dev_channels
  - **Prompt Cache (6)**: prompt_cache_1h_allowlist, prompt_cache_1h_eligible, afk_mode_header_latched, fast_mode_header_latched, cache_editing_header_latched, thinking_clear_latched
  - **Prompt (2)**: prompt_id, pending_post_compaction
  - **Error (1)**: in_memory_error_log (bounded buffer, max 100, FIFO overflow)

- REQ-2: The `Store<T>` generic must implement immutable-update semantics: `get_state()` returns a clone, `set_state(updater)` takes `FnOnce(T) -> T`. If the updated value equals the previous state (by `Eq`), skip all notifications (matching TypeScript's `Object.is` short-circuit). Otherwise, invoke the `on_change(old, new)` callback BEFORE notifying listeners. Subscribe returns an unsubscribe handle. Listeners must be deduplicated (subscribing the same listener twice is a no-op, matching TypeScript's `Set<Listener>`). Implementation must use `Arc<RwLock<T>>` for the state and a deduplicating listener collection (e.g., `IndexSet`)
- REQ-3: AppState must be the UI-layer state tree containing: settings, verbose flag, main_loop_model, tool_permission_context, tasks (HashMap), plugins (PluginState), mcp (McpState), notifications (NotificationState), repl_bridge_enabled/connected, and ~150 additional fields. It must be wrapped in `AppStateStore = Store<AppState>`
- REQ-4: The `on_change_app_state` handler must detect field-level changes between old and new AppState and trigger 6 side effects: permission mode sync (notify CCR/SDK), model selection persistence, expanded view persistence, verbose flag persistence, settings change (clear auth caches, re-apply env vars), tungsten panel visibility persistence (internal-only, guarded by user type)
- REQ-5: Session switching must be atomic — `switch_session(session_id, project_dir)` updates both fields together and emits a signal. The signal/event system must use `tokio::sync::broadcast` channel for `on_session_switch` subscribers
- REQ-6: Deferred interaction time updates must batch rapid keypresses — `update_last_interaction_time()` sets a dirty flag, `flush_interaction_time()` performs the actual `Instant::now()` write. This prevents event-loop thrashing during rapid input
- REQ-7: Scroll drain debouncing must implement: `mark_scroll_activity()` sets a flag with 150ms idle timeout, `is_scroll_draining()` checks the flag, `wait_for_scroll_idle()` polls until flag clears. Background intervals skip work while scrolling
- REQ-8: Slow operations must use lazy TTL expiry: items added with 10-second TTL and max 10 items, filtered on each get/add call. Return a static empty slice when no items remain (reference stability for UI diffing)
- REQ-9: Hook registration must merge (not overwrite): `register_hook_callbacks()` appends new hooks to existing ones, supporting multiple registration calls from SDK and plugins. `clear_registered_plugin_hooks()` removes plugin hooks but keeps SDK callback hooks
- REQ-10: All state initialization defaults must match `getInitialState()` exactly: paths use `realpathSync` with NFC normalization (fallback to raw cwd on EPERM), timestamps use `Instant::now()`, session_id uses `Uuid::new_v4()`, allowed_setting_sources defaults to `[UserSettings, ProjectSettings, LocalSettings, FlagSettings, PolicySettings]`

## Acceptance Criteria

- [ ] AC-1: SessionState singleton is accessible from any thread via `get_session_state()` without deadlock — verified by spawning 100 concurrent read/write tasks
- [ ] AC-2: `Store<T>::set_state()` notifies all subscribers exactly once per update — verified by counting listener invocations
- [ ] AC-3: `switch_session()` atomically updates both session_id and session_project_dir — verified by concurrent readers never observing mismatched pairs
- [ ] AC-4: Deferred interaction time batches 1000 rapid calls into a single timestamp write — verified by counting `Instant::now()` invocations
- [ ] AC-5: Slow operations expire after 10 seconds — verified by adding items, advancing time, and checking filtered results
- [ ] AC-6: Hook registration merges correctly: register 3 hooks, register 2 more, verify all 5 present. Clear plugin hooks, verify callback hooks remain
- [ ] AC-7: `on_change_app_state` fires correct side effects when permission_mode changes but not when unrelated fields change — verified by mock side-effect tracking
- [ ] AC-8: SessionState `getInitialState()` equivalent produces identical JSON to TypeScript's version — verified by snapshot test
- [ ] AC-9: `in_memory_error_log` never exceeds 100 entries — verified by adding 200 entries and checking length
- [ ] AC-10: `Store<T>::subscribe()` returns an unsubscribe handle that removes the listener when dropped — verified by RAII test

## Architecture

**File structure:**
```
crates/claude-state/
├── src/
│   ├── lib.rs              (pub re-exports)
│   ├── session_state.rs    (SessionState struct, 98 fields, getters/setters)
│   ├── session_init.rs     (get_initial_state(), default construction)
│   ├── app_state.rs        (AppState struct, ~150 fields)
│   ├── store.rs            (Store<T> generic with listeners)
│   ├── on_change.rs        (on_change_app_state side-effect handler)
│   ├── signals.rs          (session_switched broadcast channel, interaction time batching)
│   └── scroll.rs           (scroll drain debouncing)
```

**Concurrency model translation:**

TypeScript's SessionState is entirely single-threaded (Node.js event loop). In Rust with tokio, multiple async tasks may access state concurrently. The translation:

| TypeScript Pattern | Rust Equivalent |
|---|---|
| Module-scope `const STATE = getInitialState()` | `static STATE: LazyLock<Arc<RwLock<SessionState>>>` |
| Direct field access `STATE.foo` | `state.read().foo.clone()` via getter |
| Direct field mutation `STATE.foo = bar` | `state.write().foo = bar` via setter |
| `interactionTimeDirty` flag | `AtomicBool` (lock-free) |
| `scrollDraining` flag + setTimeout | `AtomicBool` + `tokio::time::sleep` |
| `createSignal<[SessionId]>()` | `tokio::sync::broadcast::channel` |
| `outputTokensAtTurnStart` module var | Field in SessionState (no separate module scope) |

**Key design decision:** The TypeScript code relies on single-threaded execution for atomicity (e.g., `switchSession` updates two fields "atomically" because nothing can interleave). In Rust, the `RwLock` write guard provides this guarantee — both fields are updated within the same write guard scope.

**Dependencies:**
- `tokio` (async runtime, broadcast channels, time)
- `parking_lot` (faster RwLock/Mutex than std) 
- `uuid` (SessionId generation)
- `serde` + `serde_json` (state serialization for persistence)
- `claude-types` (SessionId, AgentId, PermissionMode, etc.)

## Open Questions

### Q1: Conditional ant-only fields
**Decision**: Always include `repl_bridge_active` in the struct with a default of `false`. This matches the runtime-gating decision (USER_TYPE env var, not compile-time features) and keeps the struct simple. The field is inert when `USER_TYPE != "ant"` — no behavioral impact, just an unused `bool`.

## Out of Scope

- UI rendering logic that reads AppState (lives in `claude-tui`)
- Permission checking algorithms (lives in `claude-types` / `claude-tools`)
- Settings loading and parsing (lives in `claude-config`)
- OpenTelemetry Meter/Counter initialization (lives in `claude-analytics`, stored in SessionState as opaque handles)
