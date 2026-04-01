# Feature: Background Automation Services (MagicDocs, AutoDream, AgentSummary)

## Summary
Implement three background automation services that run during conversation idle periods: MagicDocs (automatic documentation maintenance for files with `# MAGIC DOC` headers, 13K LOC), AutoDream (periodic memory consolidation across sessions with lock-based coordination, 20K LOC), and AgentSummary (30-second periodic summarization for coordinator sub-agents, 180 LOC). All three use the shared `run_forked_agent()` pattern with cache-safe parameters and tool denial callbacks.

## Requirements

### MagicDocs
- REQ-1: Magic doc detection must match `# MAGIC DOC: [title]` at file start, optionally extracting italicized custom instructions from the next line. Registration into tracked docs map is idempotent. A FileReadTool listener auto-detects and registers magic docs when files are read
- REQ-2: Magic doc update must trigger on post-sampling hook when no tool calls in last assistant turn (conversation idle). Sequential update of all tracked docs via forked agent with Edit tool only. Update prompt focuses on terse, current-state documentation — no historical notes, no "Previously..." entries
- REQ-3: Custom prompts loadable from `~/.claude/magic-docs/prompt.md` with fallback to default. Template variables: `{{docPath}}`, `{{docContents}}`, `{{docTitle}}`, `{{customInstructions}}`

### AutoDream (Memory Consolidation)
- REQ-4: Multi-gate scheduling (cheapest check first): (1) Time gate — `(now - last_consolidated_at) >= min_hours` (one stat read on lock file mtime), (2) Scan throttle — don't scan more than every 10 minutes, (3) Session gate — count sessions touched since last consolidation >= min_sessions. Default thresholds from GrowthBook `tengu_onyx_plover`: min_hours=24, min_sessions=5
- REQ-5: Consolidation lock must use PID-based file locking at `.consolidate-lock` in memory dir: CAS-style acquire (read existing PID → check if alive and fresh <1 hour → write new PID → verify via re-read for race detection), rollback (rewind mtime, clear PID), `list_sessions_touched_since()` scans transcript dir excluding agent-*.jsonl and current session
- REQ-6: Consolidation runs a 4-phase forked agent: Phase 1 Orient (ls memory dir, read index), Phase 2 Gather (priority: daily logs → drifted memories → transcript search), Phase 3 Consolidate (merge into topic files, convert relative→absolute dates), Phase 4 Prune (keep MEMORY.md under 25KB, one-line entries ~150 chars). Tool permissions: Edit/Write to memory dir only, Bash read-only
- REQ-7: Feature gating: not KAIROS mode, not remote mode, auto-memory enabled, GrowthBook or settings enabled. Triggered from stop hooks (post-sampling)

### AgentSummary
- REQ-8: Periodic summarization at 30-second intervals (`SUMMARY_INTERVAL_MS`) for coordinator sub-agents. Summary format: 3-5 words, present tense (-ing form), include file names. Example: "Reading runAgent.ts", "Fixing null check in validate.ts"
- REQ-9: Summary generation via forked agent with all tools denied. Previous summary tracked to avoid repetition. Timer schedules on completion (not initiation) to prevent overlap. Requires >=3 messages in agent context before generating

## Acceptance Criteria

- [ ] AC-1: Magic doc header detection correctly matches valid headers and ignores invalid ones — verified by 10+ fixture tests
- [ ] AC-2: Magic doc update only triggers on idle (no tool calls in last turn) — verified by mock test
- [ ] AC-3: AutoDream gate evaluation short-circuits at cheapest gate — verified by timing test
- [ ] AC-4: Consolidation lock prevents concurrent consolidation — verified by two-process race test
- [ ] AC-5: Lock rollback correctly restores mtime on failure — verified by failure injection test
- [ ] AC-6: Agent summary produces 3-5 word present-tense descriptions — verified by filter test
- [ ] AC-7: Summary timer doesn't overlap — verified by checking single concurrent execution

## Architecture

```
crates/claude-core/src/automation/
├── mod.rs              (shared forked agent infrastructure)
├── magic_docs/
│   ├── mod.rs          (detection, registration, file read listener)
│   ├── update.rs       (post-sampling hook, sequential update)
│   └── prompts.rs      (update prompt template, custom prompt loading)
├── auto_dream/
│   ├── mod.rs          (gate evaluation, executeAutoDream)
│   ├── lock.rs         (PID-based consolidation lock, CAS acquire/rollback)
│   ├── prompt.rs       (4-phase consolidation prompt)
│   └── config.rs       (GrowthBook thresholds, feature gating)
└── agent_summary/
    ├── mod.rs          (startAgentSummarization, 30s timer)
    └── prompt.rs       (3-5 word summary prompt template)
```

## Open Questions

None.

## Out of Scope

- Memory system types/scanning (lives in `claude-memory`)
- Task registration for dream tasks (lives in `claude-tasks`)
- Forked agent execution engine (shared infrastructure in `claude-core`)
