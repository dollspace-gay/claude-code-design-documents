# Feature: Native Modules, Keybindings & Vim Mode (`claude-native` + `claude-vim` crates)

## Summary
Implement three subsystems: (1) `claude-native` — the color diff renderer (syntect-based syntax highlighting + similar-based word diff + grapheme-aware wrapping) and file index (nucleo-equivalent fuzzy search with fzf-v2 scoring), (2) keybinding system (17 contexts, 60+ actions, chord support, platform-specific defaults) integrated into `claude-tui`, and (3) `claude-vim` — a vim mode state machine supporting motions, operators, text objects, find commands, and all normal/insert mode transitions.

## Requirements

### Color Diff (`claude-native`)
- REQ-1: Syntax-highlighted diff rendering must: (1) detect language from file path/shebang, (2) highlight lines via `syntect`, (3) compute word-level diff between adjacent +/- pairs via `similar`, (4) apply background colors for added/removed/modified regions, (5) wrap lines to terminal width using grapheme-aware wrapping (`unicode-segmentation` + `unicode-width`), (6) add line numbers and +/- markers
- REQ-2: The `ColorDiff` struct takes a hunk, file path, and theme, and produces `Vec<StyledLine>` via `render(width)`

### File Index (`claude-native`)
- REQ-3: Fuzzy file search must implement nucleo/fzf-v2-equivalent scoring with constants: SCORE_MATCH=16, BONUS_BOUNDARY=8, BONUS_CAMEL=6, BONUS_CONSECUTIVE=4, BONUS_FIRST_CHAR=8, PENALTY_GAP_START=3, PENALTY_GAP_EXTENSION=1. The `FileIndex` struct stores: paths, lower_paths, char_bits (a-z bitmap per path), path_lens
- REQ-4: File index must support: `load_from_file_list(files)`, async variant `load_from_file_list_async(files)`, and `search(query, limit) → Vec<SearchResult>`. Test file paths receive a scoring penalty

### Keybindings (in `claude-tui`)
- REQ-5: The keybinding system must support 17 contexts: Global, Chat, Autocomplete, Confirmation, Help, Transcript, HistorySearch, Task, ThemePicker, Settings, Tabs, Attachments, Footer, MessageSelector, DiffDialog, ModelPicker, Select. Multiple contexts can be active simultaneously
- REQ-6: Each keybinding is a `ParsedBinding` with: keystroke sequence (chord), context, action name. Keystrokes parse: key name, ctrl, alt, shift, meta, super modifiers. Platform-specific: macOS uses Opt for Alt
- REQ-7: Chord resolution must return: Match (action found), ChordStarted (prefix match, waiting for next keystroke), ChordCancelled (invalid continuation), Unbound (no match), None (no active binding). Chords have a timeout after which the pending sequence is cancelled
- REQ-8: 60+ actions with platform-specific default bindings. User customization via `~/.claude/keybindings.json`

### Vim Mode (`claude-vim`)
- REQ-9: Vim state machine must model two top-level modes: Insert (tracking inserted_text) and Normal (with CommandState sub-states). CommandState has 11 variants: Idle, Count, Operator, OperatorCount, OperatorFind, OperatorTextObj, Find, G, OperatorG, Replace, Indent
- REQ-10: Supported motions: h, l, j, k, w, b, e, W, B, E, 0, ^, $, G, gj, gk, gg — all with optional count prefix
- REQ-11: Supported operators: d (delete), c (change), y (yank) — composable with motions, finds, and text objects. Each operator can take a count
- REQ-12: Supported text objects (16 pairs): iw, aw, iW, aW, i", a", i', a', i\`, a\`, i(, a(, i[, a[, i{, a{, i<, a< — inner and around variants
- REQ-13: Find commands: f (find forward), F (find backward), t (till forward), T (till backward) — with ; (repeat) and , (reverse repeat)
- REQ-14: Single-key commands: x (delete char), r (replace char), ~ (toggle case), J (join lines), p/P (paste after/before), D (delete to end), C (change to end), Y (yank line), o/O (open line below/above), >> (<< indent/dedent), u (undo), . (repeat last change)
- REQ-15: The `transition(state, input, ctx) → TransitionResult` function is the core state machine — it takes current state, keypress input, and mutable context, returning the new state and any actions to execute

## Acceptance Criteria

- [ ] AC-1: Color diff renders Python, Rust, and TypeScript files with correct syntax highlighting — verified by snapshot tests
- [ ] AC-2: Word-diff correctly identifies changed words within modified lines — verified by 10+ diff pair tests
- [ ] AC-3: Grapheme-aware wrapping handles CJK characters, emoji, and combining marks — verified by width calculation tests
- [ ] AC-4: File index scoring matches fzf-v2 for 20+ query/path pairs — verified by golden file tests
- [ ] AC-5: File index search returns results in < 10ms for 100,000 file paths — verified by benchmark
- [ ] AC-6: All 17 keybinding contexts can be activated/deactivated — verified by state test
- [ ] AC-7: Chord bindings resolve correctly with multi-keystroke sequences — verified by sequence tests
- [ ] AC-8: All vim motions produce correct cursor positions — verified by table-driven tests (30+ cases per motion)
- [ ] AC-9: All vim operator+motion combinations produce correct text transformations — verified by 50+ test cases
- [ ] AC-10: All 16 text object pairs select correct ranges — verified by test cases with various delimiter nesting
- [ ] AC-11: Vim state machine transitions match TypeScript's `transition()` output for 100+ input sequences — verified by golden file comparison

## Architecture

```
crates/claude-native/
├── src/
│   ├── lib.rs
│   ├── diff/
│   │   ├── color_diff.rs   (ColorDiff struct, render method)
│   │   ├── syntax.rs       (syntect theme/language detection)
│   │   ├── word_diff.rs    (similar-based word-level diff)
│   │   └── wrap.rs         (grapheme-aware line wrapping)
│   └── search/
│       ├── file_index.rs   (FileIndex struct, load, search)
│       └── scoring.rs      (fzf-v2 scoring constants and algorithm)

crates/claude-vim/
├── src/
│   ├── lib.rs
│   ├── mode.rs             (VimMode: Insert/Normal, CommandState 11 variants)
│   ├── transition.rs       (transition() state machine)
│   ├── motion.rs           (h,l,j,k,w,b,e,W,B,E,0,^,$,G,gj,gk,gg)
│   ├── operator.rs         (d,c,y with motion/find/textobj composition)
│   ├── text_object.rs      (16 pairs: iw,aw,iW,aW,i",a",i',a',etc.)
│   ├── find.rs             (f,F,t,T with ;/, repeat)
│   ├── command.rs          (x,r,~,J,p,P,D,C,Y,o,O,>>,<<,u,.)
│   └── context.rs          (TransitionContext: text buffer, cursor, clipboard)
```

**Dependencies:**
- `syntect` (syntax highlighting)
- `similar` (word-level diff)
- `unicode-segmentation` (grapheme clusters)
- `unicode-width` (display width)
- `nucleo` (or custom scoring matching fzf-v2 constants)
- `serde_json` (keybindings.json parsing)

## Open Questions

### Q1: nucleo crate vs custom scoring
**Decision**: Use the `nucleo` crate with tuned parameters. `nucleo` is the standard Rust fuzzy matching library (powers `helix` editor), well-maintained, and implements fzf-v2-compatible scoring. Configure its bonus/penalty constants to match the TypeScript values (SCORE_MATCH=16, BONUS_BOUNDARY=8, etc.) via nucleo's `MatcherConfig`. Validate parity with a golden-file test of 50+ query/path pairs extracted from the TypeScript implementation.

## Out of Scope

- Prompt input widget integration (lives in `claude-tui`, uses `claude-vim` for vim mode)
- Keybinding UI configuration commands (lives in `claude-commands`)
- Terminal input parsing (lives in `claude-tui`, crossterm events translated to vim inputs)
