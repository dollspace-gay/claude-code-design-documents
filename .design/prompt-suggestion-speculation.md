# Feature: Prompt Suggestion & Speculation Engine (`claude-core` extension)

## Summary
Implement the prompt suggestion and speculation engine — the largest undocumented subsystem at ~48K LOC. The suggestion system predicts what the user will type next (2-12 word autocomplete) using a forked agent call with cache-safe parameters. The speculation engine pre-executes the predicted user input in an isolated overlay filesystem, allowing instant "accept" that skips the full query round-trip. Together they form a pipelined prediction loop: suggest → speculate → user accepts → inject speculation results → suggest next → repeat.

## Requirements

- REQ-1: Prompt suggestion generation must use `run_forked_agent()` with a system prompt that predicts the user's next natural input (not what should happen). The forked agent must deny ALL tools via the `can_use_tool` callback (not empty tools array) to preserve prompt cache hit rates. Cache-safe parameters (effort, max_tokens, thinking) must match the parent request exactly
- REQ-2: Suggestion filtering must implement 13+ rejection categories matching `shouldFilterSuggestion()`: meta-text ("done", "nothing found"), meta-wrapped (parentheses/brackets), error messages, prefixed labels ("word:" format), too few words (<2, except slash commands and allowlist: yes/yeah/yep/ok/push/commit/deploy/check/exit/stop/continue/no), too many words (>12), too long (>=100 chars), multiple sentences, formatting (newlines, asterisks), evaluative tone ("thanks", "looks good"), Claude voice ("Let me", "I'll", "Here's")
- REQ-3: Suggestion suppression must check guards before generation: early conversation (<2 assistant turns), error in last response, cold cache (`MAX_PARENT_UNCACHED_TOKENS = 10_000`, formula: `input_tokens + cache_write_tokens + output_tokens > threshold`), rate limits, pending permissions, elicitation queue, plan mode
- REQ-4: The speculation engine must implement overlay filesystem isolation: create overlay dir at `/tmp/claude/speculation/{pid}/{id}`, copy-on-write for file edits (copy original to overlay before writing), redirect reads to overlay for written files, track written paths in a mutable `HashSet<PathBuf>`
- REQ-5: Speculation tool permissions must enforce boundaries: WRITE_TOOLS (Edit, Write, NotebookEdit) — allowed only in acceptEdits/bypassPermissions mode with copy-on-write; SAFE_READ_ONLY_TOOLS (Read, Glob, Grep, ToolSearch, LSP, TaskGet, TaskList) — allowed with overlay redirect; Bash — allow only read-only commands; all other tools — deny with boundary type. Constants: `MAX_SPECULATION_TURNS = 20`, `MAX_SPECULATION_MESSAGES = 100`
- REQ-6: Speculation completion boundaries must track 4 types: `Complete` (speculation finished naturally, with output_tokens count), `Bash` (non-read-only bash command encountered), `Edit` (tool_name + file_path of first edit), `DeniedTool` (tool_name + detail of denied tool)
- REQ-7: Speculation acceptance must: abort running speculation, copy written files from overlay to main filesystem via `copy_overlay_to_main()`, clean overlay directory, calculate time_saved_ms, strip thinking/redacted_thinking blocks from messages, remove interrupted tool_use blocks, drop whitespace-only text messages, inject cleaned messages into conversation
- REQ-8: Pipelined suggestion must generate the next suggestion while speculation is running. On acceptance, if pipelined suggestion exists and speculation completed, promote it to the active suggestion and restart speculation — forming a continuous prediction loop
- REQ-9: Analytics must log: `tengu_prompt_suggestion` (outcome: accept/ignore, similarity_ratio, time_to_accept, rejection_reason, prompt_id, generation_request_id) and `tengu_speculation` (outcome: accepted/aborted/error, duration_ms, suggestion_length, tools_executed, completed, boundary_type, time_saved_ms, is_pipelined)
- REQ-10: Feature gating must check hierarchically: env var override → GrowthBook feature flag (`tengu_chomp_inflection`) → settings toggle. Disabled in: non-interactive, swarm teammates, external rate-limited users. Speculation is ANT-only with separate config

## Acceptance Criteria

- [ ] AC-1: Suggestion generation produces 2-12 word predictions that pass all 13 filter categories — verified by 50+ test cases
- [ ] AC-2: Suggestion suppression correctly blocks generation when cache is cold — verified by injecting uncached token counts above threshold
- [ ] AC-3: Overlay filesystem isolates speculation writes — verified by checking main filesystem is unchanged after speculation
- [ ] AC-4: Copy-on-write preserves original file before overlay write — verified by diff test
- [ ] AC-5: Speculation boundaries fire for each type (Complete, Bash, Edit, DeniedTool) — verified by triggering each scenario
- [ ] AC-6: Acceptance injects cleaned messages without thinking blocks — verified by content inspection
- [ ] AC-7: Pipelined loop: suggest → accept → inject → suggest next — verified by end-to-end test
- [ ] AC-8: Overlay cleanup removes all temporary files on abort — verified by checking /tmp after abort

## Architecture

```
crates/claude-core/src/suggestion/
├── mod.rs              (shouldEnablePromptSuggestion, feature gating)
├── generate.rs         (generateSuggestion via forked agent, cache-safe params)
├── filter.rs           (shouldFilterSuggestion: 13+ rejection categories)
├── suppress.rs         (suppression guards: cache cold, rate limits, etc.)
├── analytics.rs        (logSuggestionOutcome, logSuggestionSuppressed)

crates/claude-core/src/speculation/
├── mod.rs              (SpeculationState, startSpeculation, abortSpeculation)
├── overlay.rs          (overlay filesystem: create, copy-on-write, cleanup)
├── permissions.rs      (canUseTool boundary enforcement, tool classification)
├── accept.rs           (acceptSpeculation: merge overlay, strip thinking, inject)
├── pipeline.rs         (pipelined suggestion generation, promotion)
├── boundary.rs         (CompletionBoundary 4-variant enum)
└── analytics.rs        (logSpeculation events)
```

## Open Questions

None.

## Out of Scope

- Forked agent implementation (shared infrastructure used by MagicDocs, AgentSummary, autoDream)
- UI rendering of suggestions (lives in `claude-tui`)
- AppState for PromptSuggestion/SpeculationState (lives in `claude-state`)
