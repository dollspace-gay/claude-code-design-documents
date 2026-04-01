# Feature: Terminal UI (`claude-tui` crate)

## Summary
Replace the React/Ink terminal renderer with a native Rust TUI built on `ratatui` + `crossterm` + `taffy` (flexbox layout). The TypeScript version uses a custom React reconciler (`src/ink/`) rendering to a terminal cell grid, with Yoga for flexbox layout, virtual scrolling for message lists, and ~140 React components. The Rust version must reproduce identical visual behavior: message rendering, diff viewing, permission dialogs, prompt input, spinner animation, virtual scrolling, syntax highlighting, hyperlinks, text selection, search highlighting, mouse tracking, and alternate screen mode.

## Requirements

- REQ-1: The rendering pipeline must use `ratatui` for widget rendering and `crossterm` for terminal I/O, replacing the custom Ink renderer (`src/ink/ink.tsx`, 251K lines). The pipeline: event loop → state update → layout (taffy) → render (ratatui widgets) → diff output (crossterm). Frame rate must be configurable, defaulting to 30fps matching Ink's default
- REQ-2: Layout must use `taffy` crate for flexbox computation, replacing the Yoga layout engine in `src/ink/layout/yoga.ts`. The mapping: `<Box>` → taffy flex container, `<Text>` → taffy leaf node with measured text size. All Yoga-compatible flexbox properties must be supported: flex_direction, justify_content, align_items, flex_wrap, flex_grow, flex_shrink, flex_basis, padding, margin, border, gap, min/max width/height, overflow
- REQ-3: The REPL screen must reproduce the full interactive experience from `src/screens/REPL.tsx`: message list with virtual scrolling, multiline prompt input with history, permission request modals, spinner with configurable verbs, status line, task list, agent progress, cost display, and diagnostic overlay
- REQ-4: The message rendering system must support all message types: user messages, assistant text (with markdown rendering), tool use blocks (with tool-specific formatting), tool results (with truncation indicators), thinking blocks (collapsible), system messages (info/warning/error), and progress messages
- REQ-5: The diff viewing system must reproduce `src/components/diff/` behavior: syntax-highlighted diffs with word-level change detection, line numbers, file path headers, added/removed/modified line coloring. Syntax highlighting via `syntect`, word-diff via `similar`, grapheme-aware line wrapping via `unicode-segmentation` + `unicode-width`
- REQ-6: The prompt input widget must support: multiline editing, cursor movement (home/end/word boundaries), history navigation (up/down arrow), paste handling (multi-chunk paste detection with timeout), vim mode integration (via `claude-vim` crate), placeholder text, and mask mode (for sensitive input). Must match `src/components/BaseTextInput.tsx` behavior
- REQ-7: Virtual scrolling for the message list must render only visible rows plus a buffer (matching `src/components/VirtualMessageList.tsx`). Performance target: smooth scrolling with 10,000+ messages without frame drops
- REQ-8: The keybinding system must implement 17 contexts (Global, Chat, Autocomplete, Confirmation, Help, Transcript, HistorySearch, Task, ThemePicker, Settings, Tabs, Attachments, Footer, MessageSelector, DiffDialog, ModelPicker, Select) with 60+ actions and chord support (e.g., Ctrl+K followed by Ctrl+C). Platform-specific defaults: macOS uses Opt for Alt bindings. Keybindings loaded from `~/.claude/keybindings.json`
- REQ-9: Terminal features must include: hyperlink detection and OSC 8 rendering, text selection with copy support, mouse tracking (click, hover) via crossterm mouse events, alternate screen mode for full-screen dialogs, Kitty keyboard protocol support, and ANSI 256-color/truecolor output
- REQ-10: The buddy system from `src/buddy/` (~1,298 LOC, 6 files) must be ported: a procedural companion using deterministic PRNG for animation frames
- REQ-11: The focus management system must handle pane traversal (message list, prompt input, permission dialogs, task list) with keyboard-driven focus switching, matching `src/ink/ink.tsx` FocusManager
- REQ-12: The double-buffered rendering approach from Ink must be preserved: front frame (displayed) and back frame (being rendered), with diff-based terminal output to minimize writes — matching `src/ink/terminal.ts` `writeDiffToTerminal()` and `src/ink/log-update.ts`

## Acceptance Criteria

- [ ] AC-1: The REPL renders a conversation with 10+ messages (mixed text, tool use, tool results) identically to the TypeScript version — verified by screenshot comparison
- [ ] AC-2: Diff rendering with syntax highlighting matches TypeScript output for 5 programming languages (Rust, TypeScript, Python, Go, Java) — verified by snapshot tests
- [ ] AC-3: Virtual scrolling handles 10,000 messages without exceeding 16ms frame time — verified by benchmark
- [ ] AC-4: All 17 keybinding contexts activate and deactivate correctly — verified by state machine tests
- [ ] AC-5: Chord bindings (multi-keystroke sequences) resolve correctly with timeout — verified by test with Ctrl+K, Ctrl+C sequence
- [ ] AC-6: Prompt input supports multiline editing with correct cursor positioning — verified by 20+ editing scenario tests
- [ ] AC-7: Permission dialogs render as modal overlays without disrupting the message list — verified by integration test
- [ ] AC-8: Hyperlinks render as clickable OSC 8 sequences in terminals that support them — verified by output inspection
- [ ] AC-9: Text selection and copy work in both message list and prompt input — verified by integration test
- [ ] AC-10: The TUI startup time is under 50ms to first frame — verified by benchmark
- [ ] AC-11: Taffy layout produces identical results to Yoga for the REPL's flex layout — verified by layout comparison test
- [ ] AC-12: The focus manager correctly cycles through all focusable panes — verified by automated keyboard navigation test

## Architecture

**File structure:**
```
crates/claude-tui/
├── src/
│   ├── lib.rs              (pub re-exports)
│   ├── app.rs              (App struct: event loop, state, rendering dispatch)
│   ├── screens/
│   │   ├── repl.rs         (REPL screen: message list + prompt + status)
│   │   ├── resume.rs       (ResumeConversation screen)
│   │   └── doctor.rs       (Doctor diagnostic screen)
│   ├── widgets/
│   │   ├── message_list.rs (VirtualMessageList with scroll)
│   │   ├── prompt_input.rs (multiline input with history/paste/vim)
│   │   ├── permission.rs   (PermissionRequest modal dialog)
│   │   ├── spinner.rs      (animated spinner with configurable verbs)
│   │   ├── status_line.rs  (bottom status bar)
│   │   ├── task_list.rs    (TaskListV2 widget)
│   │   ├── diff_view.rs    (syntax-highlighted diff rendering)
│   │   ├── agent_progress.rs (agent status and progress)
│   │   ├── cost_display.rs (token/cost tracking display)
│   │   └── select.rs       (generic select/menu widget)
│   ├── layout/
│   │   ├── mod.rs          (taffy-based flexbox layout engine)
│   │   └── geometry.rs     (size calculations, text measurement)
│   ├── rendering/
│   │   ├── markdown.rs     (markdown → ratatui spans)
│   │   ├── syntax.rs       (syntect-based code highlighting)
│   │   ├── diff.rs         (word-diff + line-number rendering)
│   │   ├── ansi.rs         (ANSI escape code parsing/rendering)
│   │   └── frame.rs        (double-buffered Frame with diff output)
│   ├── input/
│   │   ├── keybindings.rs  (17 contexts, 60+ actions, chord resolver)
│   │   ├── mouse.rs        (click, hover, scroll handling)
│   │   └── focus.rs        (FocusManager for pane traversal)
│   ├── features/
│   │   ├── hyperlinks.rs   (OSC 8 hyperlink rendering)
│   │   ├── selection.rs    (text selection + copy)
│   │   ├── search.rs       (search highlighting overlay)
│   │   └── buddy.rs        (procedural companion, PRNG animation)
│   └── terminal/
│       ├── mod.rs          (Terminal abstraction over crossterm)
│       ├── capabilities.rs (feature detection: truecolor, kitty, mouse)
│       └── alt_screen.rs   (alternate screen mode management)
```

**Component mapping (React/Ink → ratatui):**

| React/Ink Pattern | Rust/Ratatui Equivalent |
|---|---|
| `<Box>` | `taffy` flex container + `ratatui::layout::Rect` |
| `<Text>` | `ratatui::text::Line` / `Span` |
| `useState()` | Struct field mutation |
| `useEffect()` | Tokio task or channel receiver |
| `useSyncExternalStore()` | Store subscription callback |
| `useInput()` | crossterm `Event::Key` handler |
| `useTerminalSize()` | crossterm `terminal::size()` |
| React reconciler | Direct widget rendering per frame |
| Ink's `FiberRoot` | ratatui `Frame` + `Buffer` |
| `measureElement()` | `taffy::compute_layout()` + text measurement |
| `log-update` diff output | crossterm `write!` with cursor positioning |

**Event loop architecture:**
```rust
loop {
    // 1. Poll for events (keyboard, mouse, resize, signals)
    let event = crossterm::event::poll(Duration::from_millis(33))?; // ~30fps
    
    // 2. Dispatch to active screen's event handler
    if let Some(event) = event {
        keybinding_system.resolve(event, active_contexts);
        // → updates app state
    }
    
    // 3. Check for state changes from background tasks (query stream, tool execution)
    while let Ok(msg) = state_rx.try_recv() {
        app_state.apply(msg);
    }
    
    // 4. Layout with taffy
    taffy.compute_layout(root_node, available_size);
    
    // 5. Render with ratatui
    terminal.draw(|frame| {
        active_screen.render(frame, &app_state, &layout);
    })?;
}
```

**Dependencies:**
- `ratatui` (terminal widgets and rendering)
- `crossterm` (terminal I/O, events, raw mode)
- `taffy` (flexbox layout computation)
- `syntect` (syntax highlighting for diffs and code)
- `similar` (word-level diff computation)
- `unicode-segmentation` (grapheme cluster handling)
- `unicode-width` (display width calculation)
- `claude-state` (AppState, Store subscriptions)
- `claude-native` (color diff, file index)
- `claude-vim` (vim mode state machine)

## Open Questions

### Q1: Image rendering in terminal
**Decision**: Support inline image rendering via `ratatui-image` crate, matching Claude Code's existing iTerm2/Kitty protocol support. Feature-detect terminal capabilities at startup (query TERM, TERM_PROGRAM env vars and CSI responses) and fall back to `open` crate for external viewer in unsupported terminals.

### Q2: React hook equivalent for state subscriptions
**Decision**: Re-render all widgets every frame (option a). Ratatui is an immediate-mode rendering library — this is its native pattern. At 30fps with the diff-based terminal output, only changed cells are written. The cost of re-rendering widgets that haven't changed is negligible compared to the complexity of subscription plumbing. This matches how every major ratatui application works and keeps the codebase simple.

## Out of Scope

- Tool-specific result rendering logic (each tool's rendering is part of `claude-tools`, invoked via the `ToolRenderer` trait)
- Voice input/output integration (feature-gated, minimal scope in TypeScript)
- Desktop/IDE-specific features (VS Code, JetBrains extensions interact via bridge, not TUI)
- Output styles loading (lives in `claude-config`)
