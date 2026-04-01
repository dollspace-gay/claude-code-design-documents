# Feature: Memory System (`claude-memory` crate)

## Summary
Implement the persistent memory system that allows Claude Code to retain context across sessions. The system manages four memory types (User, Feedback, Project, Reference) stored as Markdown files with YAML frontmatter in `~/.claude/projects/{sanitized-git-root}/memory/`. It includes memory scanning, relevance-based selection (using Sonnet to choose which memories to surface), team memory with security-hardened path validation, and memory extraction from conversations.

## Requirements

- REQ-1: Memory directory structure must follow `~/.claude/projects/{sanitized-git-root}/memory/` with optional team subdirectory `{memory_dir}/team/` (when TEAMMEM feature enabled). Memory files are Markdown with YAML frontmatter containing name, description, type (User/Feedback/Project/Reference), and content body
- REQ-2: Four memory types form a closed taxonomy: User (role, goals, preferences), Feedback (guidance on approach — what to avoid/repeat), Project (ongoing work, initiatives, deadlines), Reference (pointers to external systems). Each type has distinct save triggers and usage patterns as documented in the system prompt
- REQ-3: Memory scanning must discover all `.md` files in the memory directory, parse YAML frontmatter to extract headers (name, description, type), and track modification time. The `scan_memory_files(dir) → Vec<MemoryHeader>` function must handle malformed frontmatter gracefully (skip files, don't crash)
- REQ-4: Relevance-based selection must use a model call (Sonnet) to determine which memories are relevant to the current query: `find_relevant_memories(query, memory_dir, already_surfaced) → Vec<RelevantMemory>`. The `already_surfaced` set prevents re-selecting memories that are already in context
- REQ-5: Team memory writes must implement two-pass path validation to prevent directory traversal attacks: Pass 1 — `resolve()` containment check (is the resolved path within the memory directory?), Pass 2 — `realpath()` on deepest existing ancestor + symlink check. Must reject: null bytes, URL-encoded traversals (`%2e%2e`), Unicode normalization attacks, symlink escapes
- REQ-6: The MEMORY.md index file must be maintained: one-line entries under 150 characters each, pointers to individual memory files, loaded into conversation context automatically. Lines after 200 are truncated
- REQ-7: Memory extraction from conversations must detect when the model writes memory files and validate the content/format matches the expected structure (frontmatter + content, correct type field, no placeholder text)

## Acceptance Criteria

- [ ] AC-1: Memory files round-trip: write → scan → read produces identical content — verified by roundtrip test
- [ ] AC-2: Scanning handles 100+ memory files without performance degradation — verified by benchmark
- [ ] AC-3: Team memory path validation rejects all traversal attack vectors — verified by 20+ attack pattern tests (null bytes, `../../`, `%2e%2e`, symlinks, Unicode normalization)
- [ ] AC-4: Malformed frontmatter (missing fields, invalid YAML) is skipped gracefully — verified by fixture test
- [ ] AC-5: MEMORY.md stays under 200 lines after 50 memory additions — verified by stress test
- [ ] AC-6: Relevance selection returns only memories not in `already_surfaced` — verified by mock model test

## Architecture

```
crates/claude-memory/
├── src/
│   ├── lib.rs
│   ├── types.rs            (MemoryType enum, MemoryFile, MemoryHeader, RelevantMemory)
│   ├── scan.rs             (scan_memory_files: discover and parse .md files)
│   ├── select.rs           (find_relevant_memories: model-based selection)
│   ├── index.rs            (MEMORY.md management: add, remove, update entries)
│   ├── extract.rs          (memory extraction from conversation)
│   ├── team/
│   │   ├── mod.rs          (team memory operations)
│   │   └── security.rs     (two-pass path validation, traversal rejection)
│   ├── paths.rs            (sanitized git root → memory dir path)
│   └── error.rs            (MemoryError enum)
```

**Dependencies:**
- `serde_yml` (YAML frontmatter parsing)
- `pulldown-cmark` (Markdown parsing)
- `globset` (file discovery)
- `claude-types` (MemoryType enum)

## Open Questions

None.

## Out of Scope

- System prompt injection of memories (lives in `claude-core/context.rs`)
- Memory-related model calls (the select function calls the API, but the actual API client lives in `claude-core`)
- Session memory service (persistence of session-specific memories lives in services layer)
