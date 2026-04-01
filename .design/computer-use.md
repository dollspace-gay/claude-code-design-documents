# Feature: Computer Use (Vision/Control)

## Summary
Implement the computer use subsystem (codenamed "Chicago") that enables Claude to see and interact with the user's screen via MCP server integration. This includes 13 files covering feature gates (GrowthBook-based, subscription-gated), MCP server setup, a CLI executor wrapping Swift and Rust native binaries, coordinate mode configuration (pixels), and supporting infrastructure (escape hotkey, input loading, app locking, cleanup). The system is gated behind Max/Pro subscriptions.

## Requirements

- REQ-1: Feature gating via GrowthBook `tengu_malort_pedway` config must check: Max/Pro subscription (unless ANT bypass), not monorepo dev config. Sub-gates: pixel_validation, clipboard_paste_multiline, mouse_animation, hide_before_action, auto_target_display, clipboard_guard. Coordinate mode frozen at first read (default: "pixels")
- REQ-2: MCP server setup via `setup_computer_use_mcp()` must generate dynamic MCP config with allowed tools list for the computer use MCP server
- REQ-3: CLI executor must wrap platform-native binaries (Swift on macOS, Rust on Linux) implementing the `ComputerExecutor` interface for screen capture, mouse/keyboard input
- REQ-4: Supporting infrastructure: escape hotkey handler (interrupt computer use), input event loading, computer use lock (prevent concurrent sessions), cleanup on session end, application name resolution for targeting

## Acceptance Criteria

- [ ] AC-1: Feature gates correctly block non-Max/Pro users — verified by subscription mock test
- [ ] AC-2: MCP config generation produces valid server config — verified by schema validation
- [ ] AC-3: Coordinate mode freezes after first access — verified by mutation test
- [ ] AC-4: Computer use lock prevents concurrent sessions — verified by race test

## Architecture

```
crates/claude-utils/src/computer_use/
├── mod.rs              (setup, MCP config generation)
├── gates.rs            (ChicagoConfig, subscription check, sub-gates)
├── executor.rs         (ComputerExecutor: Swift/Rust native wrapper)
├── input.rs            (input event loading)
├── hotkey.rs           (escape hotkey handler)
├── lock.rs             (concurrent session prevention)
└── cleanup.rs          (session cleanup)
```

## Open Questions

None.

## Out of Scope

- Native Swift/Rust binary implementations (separate build artifacts)
- MCP server protocol (lives in `claude-mcp`)
- Screen capture algorithms (in native binaries)
