# Feature: CLI Entry Points & Commands (`claude-cli` + `claude-commands` crates)

## Summary
Implement the binary entry point and ~103 command implementations. The `claude-cli` crate is the binary: clap-based CLI parsing, TUI bootstrap, parallel prefetch (MDM, keychain, feature flags), and routing to headless/interactive/bridge modes. The `claude-commands` crate is a library containing all ~103 command implementations across three types: Local (sync text), LocalJsx (TUI rendering), and Prompt (generates content for model execution). Commands are split ~75 public + ~28 internal-only.

## Requirements

- REQ-1: The main entry point must follow the TypeScript startup sequence: (1) profile checkpoints, (2) parse CLI args via clap, (3) run 11 migrations, (4) initialize bootstrap state/permissions/models/settings, (5) setup hooks/worktrees/background jobs, (6) route to: headless query (piped input or `--print`), interactive REPL, or bridge mode (IDE extension)
- REQ-2: Parallel prefetch at startup must fire concurrently before heavy initialization: MDM settings read (macOS/Windows), keychain token prefetch (OAuth + API key), and GrowthBook feature flag fetch. These must complete or timeout before the main initialization path needs their results
- REQ-3: The command system must support 3 command types: Local (sync execution returning text/compact/skip result), LocalJsx (TUI rendering via callback, Rust equivalent: renders ratatui widgets), Prompt (generates content blocks for model execution with allowed_tools, progress_message, and model override)
- REQ-4: All ~75 public commands must be implemented: addDir, advisor, agents, branch, btw, chrome, clear, color, compact, config, copy, desktop, context, cost, diff, doctor, effort, exit, fast, feedback, files, heapDump, help, ide, init, installGitHubApp, installSlackApp, keybindings, mcp, memory, mobile, model, outputStyle, plan, plugin, pr_comments, rateLimitOptions, releaseNotes, reloadPlugins, remoteEnv, rename, resume, review, rewind, securityReview, session, skills, stats, status, statusline, stickers, tag, terminalSetup, theme, ultrareview, upgrade, usage, usageReport, vim, and feature-gated public commands (voice, buddy, proactive, brief, assistant, bridge, workflows, ultraplan, etc.)
- REQ-5: All ~28 internal-only commands must be behind `#[cfg(feature = "internal")]`: backfillSessions, breakCache, bughunter, commit, commitPushPr, ctx_viz, goodClaude, issue, mockLimits, bridgeKick, version, resetLimits, onboarding, share, summary, teleport, antTrace, perfIssue, env, oauthRefresh, debugToolCall, and others
- REQ-6: CLI argument parsing via clap must support: positional prompt argument, `--model`, `--effort`, `--print` (headless), `--resume`/`--continue` (session restore), `--verbose`, `--add-dir`, `--settings`, `--plugin-dir`, `--allowedTools`, `--dangerouslySkipPermissions`, and all other flags from TypeScript's Commander.js definition
- REQ-7: Session restore (`--resume`, `--continue`, `/resume` command) must load TypeScript JSONL transcripts and reconstruct the message history for continuation. This is critical for cross-version compatibility
- REQ-8: The `/init` command must implement multi-phase CLAUDE.md generation: explore codebase, identify key files and patterns, generate structured project documentation
- REQ-9: The `/commit` and `/commit-push-pr` commands must produce git commits with proper attribution format matching the TypeScript version

## Acceptance Criteria

- [ ] AC-1: `claude --version` prints version info and exits — verified by running binary
- [ ] AC-2: `claude --print "hello"` executes headless query and prints result — verified by capturing stdout
- [ ] AC-3: `claude` with no args launches interactive REPL — verified by PTY test
- [ ] AC-4: All public commands appear in `/help` output — verified by assertion test
- [ ] AC-5: Internal-only commands are absent when built without `internal` feature — verified by building without feature and checking command list
- [ ] AC-6: Session resume loads TypeScript-generated JSONL transcripts — verified by loading 5 real transcript files
- [ ] AC-7: Startup parallel prefetch completes within 2 seconds — verified by benchmark
- [ ] AC-8: `/init` generates a valid CLAUDE.md — verified by running against a test repository
- [ ] AC-9: `/commit` produces a git commit with correct attribution — verified by checking commit message format

## Architecture

```
crates/claude-cli/
├── src/
│   ├── main.rs             (entry point, profiling, clap parsing, routing)
│   ├── args.rs             (clap derive definitions for CLI args)
│   ├── bootstrap.rs        (parallel prefetch, initialization, setup)
│   ├── headless.rs         (--print mode: single query, stdout output)
│   ├── repl.rs             (interactive REPL: delegates to claude-tui)
│   ├── bridge.rs           (bridge mode: delegates to claude-bridge)
│   └── print/
│       ├── mod.rs          (PrintOrchestrator: structured/plain output)
│       ├── structured_io.rs (JSON output for SDK consumers)
│       └── transport.rs    (stdout/file output transports)

crates/claude-commands/
├── src/
│   ├── lib.rs              (command registry, COMMANDS() + INTERNAL_ONLY_COMMANDS)
│   ├── git/
│   │   ├── commit.rs       (/commit with attribution)
│   │   ├── commit_push_pr.rs
│   │   └── branch.rs
│   ├── session/
│   │   ├── resume.rs       (/resume, /continue — JSONL loading)
│   │   ├── compact.rs      (/compact — manual compaction)
│   │   └── export.rs       (/export — session export)
│   ├── config/
│   │   ├── model.rs        (/model — model selection)
│   │   ├── effort.rs       (/effort — effort level)
│   │   ├── fast.rs         (/fast — toggle fast mode)
│   │   ├── theme.rs        (/theme — color theme)
│   │   └── permissions.rs  (/permissions — permission mode)
│   ├── project/
│   │   ├── init.rs         (/init — CLAUDE.md generation)
│   │   └── doctor.rs       (/doctor — diagnostics)
│   ├── extensions/
│   │   ├── mcp.rs          (/mcp — MCP server management)
│   │   ├── plugin.rs       (/plugin — plugin marketplace)
│   │   └── skills.rs       (/skills — skill listing)
│   ├── auth/
│   │   ├── login.rs        (/login — OAuth flow)
│   │   └── logout.rs       (/logout — token removal)
│   ├── ui/
│   │   ├── help.rs         (/help)
│   │   ├── clear.rs        (/clear)
│   │   ├── status.rs       (/status, /stats, /cost)
│   │   └── vim.rs          (/vim — toggle vim mode)
│   ├── review/
│   │   ├── review.rs       (/review)
│   │   └── security.rs     (/security-review)
│   └── internal/           (#[cfg(feature = "internal")])
│       ├── backfill.rs
│       ├── bughunter.rs
│       ├── debug.rs
│       └── teleport.rs
```

**Dependencies:**
- `clap` (derive API for CLI argument parsing)
- `claude-core` (query engine for headless mode)
- `claude-tui` (REPL screen for interactive mode)
- `claude-bridge` (bridge mode for IDE extensions)
- `claude-config` (settings, migrations)
- `claude-auth` (OAuth for /login)
- `claude-state` (SessionState, AppState)
- `claude-types` (Command enum)

## Open Questions

None.

## Out of Scope

- REPL rendering (lives in `claude-tui`)
- Query execution (lives in `claude-core`)
- Tool implementations (lives in `claude-tools`)
- Bridge protocol (lives in `claude-bridge`)
