# Feature: Query Engine & Streaming (`claude-core` + `claude-query` crates)

## Summary
Implement the query orchestration engine — the central loop that drives all LLM interactions in Claude Code. The query function is an async generator that: builds system prompts from context (git, CLAUDE.md, memories), sends streaming requests to the Anthropic API, parses tool_use blocks, runs permission checks, executes tools (parallel for reads, serial for writes), feeds results back, auto-compacts when approaching token budget, and loops until `stop_reason != tool_use`. This is split across `claude-core` (query engine, tool loop, streaming) and `claude-query` (config, deps, stop hooks, token budget).

## Requirements

- REQ-1: The `query()` function must be an async stream (Rust `Stream<Item = QueryEvent>`) that yields events matching the TypeScript async generator signature: `StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage`. The function takes `QueryParams` containing: messages, system_prompt, user_context, system_context, can_use_tool callback, tool_use_context, fallback_model, query_source, max_output_tokens_override, max_turns, skip_cache_write, task_budget, and deps (injectable dependency object)
- REQ-2: Internal loop state must track: messages (mutable across iterations), tool_use_context, auto_compact_tracking, max_output_tokens_recovery_count, has_attempted_reactive_compact, max_output_tokens_override, pending_tool_use_summary, stop_hook_active, turn_count, and transition. State is destructured at the top of each iteration; immutable params (system_prompt, user_context, etc.) are separated from mutable state
- REQ-3: The streaming event pipeline must support: RequestStart (with request_id), TextDelta (incremental text), ThinkingDelta (incremental thinking), ToolUseStart (id + name), ToolUseInput (partial JSON for progressive rendering), ToolResult (id + result), Message (complete message), Error (API error), Done (stop_reason + usage)
- REQ-4: Context compaction must implement a 4-stage pipeline in order: (1) Snip compaction (feature-gated `history-snip`, tracks snip_tokens_freed), (2) Microcompaction (cached tool-use reduction via deps.microcompact(), with optional pending cache edits gated on `cached-microcompact`), (3) Context collapse (projects collapsed view from store, gated on `context-collapse`, non-yielding), (4) Autocompaction (main compaction via deps.autocompact(), receives pre/post token counts and compaction cost metadata)
- REQ-5: Token budget tracking must implement `BudgetTracker` with fields: continuation_count, last_delta_tokens, last_global_turn_tokens, started_at. The `check_token_budget()` function returns Continue (with nudge_message and completion percentage) or Stop (with optional completion_event). Budget thresholds: completion at 90% (`COMPLETION_THRESHOLD = 0.9`), diminishing returns detection at continuation_count >= 3 with delta_tokens < 500
- REQ-6: Tool execution within the loop must support: parallel execution for concurrency-safe read-only tools, serial execution for write tools, abort/cancellation via `AbortController` equivalent (tokio `CancellationToken`), permission checking before each execution, pre-tool-use and post-tool-use hook invocation (via injected `HookRunner` trait), result truncation for oversized outputs, and disk persistence for large results with preview
- REQ-7: The query dependency injection (`QueryDeps`) must be a trait with methods: `autocompact()`, `microcompact()`, `make_api_request()`, `count_tokens()`, and `check_stop_hooks()` — allowing test doubles to be injected for unit testing the query loop without real API calls
- REQ-8: The tool loop must continue iterating until `stop_reason != tool_use` or `max_turns` is reached. Each iteration: (1) yield the assistant message, (2) extract tool_use blocks, (3) for each tool_use: validate input against schema → check permissions → execute tool → handle result, (4) collect all tool results as a user message, (5) re-invoke the API with updated messages
- REQ-9: Cross-iteration budget tracking must maintain `task_budget_remaining` as a loop-local variable (not in State), decremented after each API response by tokens consumed. When budget exhausted, emit a Stop decision
- REQ-10: The Anthropic SDK streaming client must handle: SSE event parsing, partial JSON accumulation for tool_use inputs, usage tracking (input_tokens, output_tokens, cache_read_input_tokens, cache_creation_input_tokens), web_search_requests counting, and retry with exponential backoff on transient errors (429, 500, 503)

## Acceptance Criteria

- [ ] AC-1: The query stream produces events in the correct order for a single-turn response: RequestStart → TextDelta* → Done — verified by mock API server test
- [ ] AC-2: The query stream handles a multi-turn tool-use flow: RequestStart → ToolUseStart → ToolUseInput* → ToolResult → RequestStart (re-invoke) → TextDelta* → Done — verified by mock test with 3+ tool uses
- [ ] AC-3: Auto-compaction triggers when token usage exceeds 90% of budget — verified by test that injects messages totaling 95% of budget
- [ ] AC-4: Parallel tool execution runs concurrency-safe tools simultaneously — verified by timing test showing 3 read-only tools complete in ~1x single-tool time, not 3x
- [ ] AC-5: Serial tool execution prevents concurrent writes — verified by test showing 2 write tools execute in sequence
- [ ] AC-6: Token budget tracking detects diminishing returns (continuation_count >= 3, delta < 500) and emits Stop — verified by unit test
- [ ] AC-7: The QueryDeps trait can be fully mocked for testing — verified by a test that uses test doubles for all 5 methods
- [ ] AC-8: Abort/cancellation stops the tool loop mid-execution — verified by test that cancels after first tool use
- [ ] AC-9: Oversized tool results are truncated and persisted to disk — verified by test with 200KB tool output exceeding max_result_size_chars
- [ ] AC-10: API retry with exponential backoff handles 429/500/503 errors — verified by mock server returning errors then succeeding

## Architecture

**File structure:**
```
crates/claude-core/
├── src/
│   ├── lib.rs
│   ├── query/
│   │   ├── mod.rs          (pub query() async stream function)
│   │   ├── state.rs        (internal State struct, transitions)
│   │   ├── streaming.rs    (SSE parser, partial JSON accumulator, event types)
│   │   ├── tool_loop.rs    (tool execution: parallel/serial, permission check, result handling)
│   │   ├── compaction.rs   (4-stage compaction pipeline: snip → micro → collapse → auto)
│   │   └── api_client.rs   (Anthropic SDK wrapper: streaming, retry, usage tracking)
│   ├── coordinator.rs      (multi-agent orchestration mode)
│   └── context.rs          (system prompt assembly, CLAUDE.md loading)

crates/claude-query/
├── src/
│   ├── lib.rs
│   ├── config.rs           (QueryConfig builder)
│   ├── deps.rs             (QueryDeps trait + production implementation)
│   ├── token_budget.rs     (BudgetTracker, check_token_budget, COMPLETION_THRESHOLD)
│   └── stop_hooks.rs       (stop hook checking logic)
```

**Key design decisions:**

The TypeScript `query()` is an `async function*` (async generator). Rust doesn't have native async generators, so we use `async-stream` crate or manual `Stream` implementation via `poll_next`. The `async-stream` crate provides a `stream! { }` macro that closely mirrors the TypeScript `yield` pattern.

The tool loop inside the query function uses `tokio::JoinSet` for parallel tool execution. Tools tagged `is_concurrency_safe() == true` are spawned concurrently; others run sequentially. The `CancellationToken` from `tokio-util` replaces `AbortController`.

The streaming SSE parser accumulates partial JSON for `tool_use` input blocks. TypeScript uses string concatenation; Rust should use a `Vec<u8>` buffer with `serde_json::from_slice` at completion.

**Dependencies:**
- `tokio` (async runtime, JoinSet, CancellationToken)
- `tokio-stream` (Stream trait, stream combinators)
- `async-stream` (stream! macro for generator-like syntax)
- `reqwest` (HTTP streaming for SSE)
- `serde_json` (JSON parsing for tool inputs/outputs)
- `claude-types` (Message, Tool trait, StreamEvent, etc.)
- `claude-state` (SessionState access for cost/token tracking)

## Open Questions

### Q1: Streaming backpressure
**Decision**: Use a bounded `tokio::sync::mpsc` channel (capacity 256) between the query stream producer and the UI consumer. When the buffer fills, the producer awaits (natural backpressure, matching the TypeScript async generator's pull-based semantics). TextDelta events are NOT dropped — all events are delivered in order, the producer just slows down. This prevents unbounded memory growth while preserving correctness.

## Out of Scope

- Individual tool implementations (those live in `claude-tools`)
- System prompt content (static text lives in `claude-cli` constants, template logic in `claude-core/context.rs`)
- MCP tool registration (lives in `claude-mcp`, injected via trait)
- Hook execution engine (lives in `claude-hook-engine`, injected via `HookRunner` trait)
- UI rendering of stream events (lives in `claude-tui`)
