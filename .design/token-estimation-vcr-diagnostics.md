# Feature: Token Estimation, VCR, & Diagnostic Tracking

## Summary
Implement three supporting services: token estimation (496 LOC — API-based counting with Haiku fallback and rough estimation), VCR test fixture recording/playback (406 LOC — SHA1-hashed fixture management for deterministic API testing), and diagnostic tracking (397 LOC — LSP-based language diagnostic tracking for file edit quality monitoring).

## Requirements

### Token Estimation (in `claude-core`)
- REQ-1: Token counting must use a 3-tier fallback: (1) API-based via `count_messages_tokens_with_api()` routing to Bedrock or Anthropic, (2) Haiku-based fallback via `count_tokens_via_haiku_fallback()` using Haiku 4.5 (or Sonnet for Vertex global region / thinking blocks), (3) rough estimation via `rough_token_count_estimation(content, bytes_per_token=4)` with file-type awareness (JSON: 2 bytes/token, default: 4)
- REQ-2: Per-block estimation must handle: text (full estimation), image/document (2000 tokens conservative), tool_use (name + JSON input), tool_result (recursive content), thinking (full text), other (JSON stringified). Constants: `TOKEN_COUNT_THINKING_BUDGET = 1024`, `TOKEN_COUNT_MAX_TOKENS = 2048`
- REQ-3: Haiku fallback model selection: Vertex global → Sonnet, Bedrock+thinking → Sonnet, Vertex+thinking → Sonnet, otherwise → Haiku. Strip `tool_reference` blocks before sending (not valid without `tool_search` beta)

### VCR Test Infrastructure (in `claude-utils` or test support)
- REQ-4: VCR fixture management via `with_fixture(input, fixture_name, f)` using SHA1 hash of input for deterministic naming. Record mode controlled by `VCR_RECORD` env var. Fixtures stored at configurable root (`CLAUDE_CODE_TEST_FIXTURES_ROOT`)
- REQ-5: Dehydration/hydration for cross-platform portability: replace dynamic values with tokens (`[NUM]`, `[DURATION]`, `[COST]`, `[CONFIG_HOME]`, `[CWD]`, `[COMMANDS]`). Windows path handling (forward slashes, JSON escaping)
- REQ-6: Streaming VCR variant (`with_streaming_vcr`) must record/playback async generator output. Token count VCR variant (`with_token_count_vcr`) caches count results

### Diagnostic Tracking (in `claude-tools` or `claude-tui`)
- REQ-7: LSP-based diagnostic tracking must: capture baseline diagnostics before file edit (`before_file_edited`), compute delta after edit (`get_new_diagnostics`), format summary (4000 char max). Multi-protocol support: `file://`, `_claude_fs_right:`, `_claude_fs_left:` (split diff view)
- REQ-8: Severity symbols: Error (cross), Warning (warning sign), Info (info), Hint (lightbulb). Last processed timestamp tracking to detect only NEW diagnostics

## Acceptance Criteria

- [ ] AC-1: Token counting falls back correctly through 3 tiers — verified by disabling each tier
- [ ] AC-2: Rough estimation produces reasonable counts for JSON vs text — verified by known-length fixtures
- [ ] AC-3: VCR fixtures are deterministic (same input → same fixture) — verified by hash stability test
- [ ] AC-4: VCR dehydration/hydration round-trips across platforms — verified by Windows path test
- [ ] AC-5: Diagnostic tracking detects new errors after file edit — verified by LSP mock test
- [ ] AC-6: Diagnostic summary stays under 4000 chars — verified by large diagnostic set test

## Architecture

Token estimation integrates into `claude-core` (query engine uses it for budget tracking). VCR is test infrastructure in `claude-utils`. Diagnostic tracking integrates with the LSP tool in `claude-tools`.

## Open Questions

None.

## Out of Scope

- Actual LSP server management (lives in `claude-tools` LSPTool)
- Token budget tracking (lives in `claude-query`)
