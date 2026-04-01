# Feature: Auxiliary Services & Utilities

## Summary
Implement remaining auxiliary services and utilities that support the main subsystems: Claude-in-Chrome integration (6 files — native host communication and MCP server for browser automation), suggestion utilities (5 files — Slack channels, directory completion, shell history, command suggestions, skill usage tracking), attachment handling (file/image processing for conversation input), session history management (pasted content, history entries), cost tracking (token/USD accumulation), project onboarding state (step tracking for new projects), and sandbox utilities (environment adaptation).

## Requirements

### Claude-in-Chrome (`claude-utils`)
- REQ-1: Chrome extension integration must support: native host communication protocol, MCP server for browser automation, prompt building for Chrome context, configuration setup (standard + portable variants)

### Suggestion Utilities (`claude-utils`)
- REQ-2: Input suggestion sources must include: Slack channel autocompletion, file path/directory completion, shell history-based suggestions, command autocompletion, and skill usage tracking for ranking. These feed into the TUI's typeahead system

### Attachment Handling (`claude-utils`)
- REQ-3: Attachment processing must handle: image files (with media type detection), text file content (with size limits), pasted content (inline for <1024 chars, hash reference for larger), and content block construction for API messages

### Session History (`claude-utils`)
- REQ-4: Session history must manage: max 100 history entries, pasted content store (referenced by ID), reference parsing (`[Pasted text #N +M lines]`, `[Image #N]`), reference expansion (inline content from paste store). Paste reference format: `format_pasted_text_ref(id, num_lines)`, `format_image_ref(id)`

### Cost Tracking (`claude-state` extension)
- REQ-5: Cost tracking must accumulate: total USD, total API duration, per-model token breakdown (input, output, cache_read, cache_creation), web search request count. Support session cost persistence and restoration via `get_stored_session_costs(session_id)`

### Project Onboarding (`claude-config` extension)
- REQ-6: Project onboarding must track steps: (1) "Create new app or clone" (enabled when dir empty), (2) "Run /init for CLAUDE.md" (enabled when dir not empty). Memoized `should_show_project_onboarding()` check, seen-count tracking, completion caching

### Sandbox (`claude-utils`)
- REQ-7: Sandbox utilities must provide: environment adapter for sandbox detection, UI utilities for sandbox-specific interactions

## Acceptance Criteria

- [ ] AC-1: Chrome native host communication sends/receives valid messages — verified by mock test
- [ ] AC-2: Directory completion produces valid file paths — verified by filesystem test
- [ ] AC-3: Pasted content >1024 chars uses hash reference, <1024 inlines — verified by size test
- [ ] AC-4: History stays at max 100 entries — verified by overflow test
- [ ] AC-5: Cost tracking round-trips through persistence — verified by save/load test
- [ ] AC-6: Onboarding steps reflect directory state — verified by empty vs non-empty dir test

## Architecture

```
crates/claude-utils/src/
├── chrome/
│   ├── native_host.rs  (Chrome native messaging protocol)
│   ├── mcp_server.rs   (MCP server for browser automation)
│   ├── prompt.rs       (Chrome context prompt building)
│   └── setup.rs        (configuration, portable variant)
├── suggestions/
│   ├── slack.rs        (Slack channel completion)
│   ├── directory.rs    (file path completion)
│   ├── shell_history.rs (shell history suggestions)
│   ├── commands.rs     (command autocompletion)
│   └── skill_usage.rs  (skill ranking by usage)
├── attachments.rs      (file/image processing, content blocks)
├── history.rs          (session history, paste store, references)
├── sandbox.rs          (environment adapter, UI utils)

crates/claude-state/src/
├── cost_tracker.rs     (USD/token accumulation, persistence)

crates/claude-config/src/
├── onboarding.rs       (project onboarding steps, completion tracking)
```

## Open Questions

None.

## Out of Scope

- TUI typeahead widget (lives in `claude-tui`, consumes suggestion utilities)
- Chrome extension itself (separate browser extension project)
- Sandbox creation/management (handled by container runtime)
