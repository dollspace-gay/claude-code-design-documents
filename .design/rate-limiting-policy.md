# Feature: Rate Limiting & Policy Enforcement (`claude-core` / `claude-config` extension)

## Summary
Implement the unified rate limiting and policy enforcement system, currently scattered across 5+ TypeScript files (~3.5K LOC). This covers Claude.ai usage limit tracking (5-hour and 7-day windows with early warnings), rate limit message generation (subscription-aware with upsell guidance), mock infrastructure for testing (19 scenarios, ANT-only), and enterprise policy limits (organization-level feature restrictions with ETag-cached HTTP polling).

## Requirements

- REQ-1: Rate limit state tracking must parse unified headers (`anthropic-ratelimit-unified-*`) from API responses, extracting: status, utilization percentage, reset time, overage info. Types: `RateLimitType` (five_hour, seven_day, seven_day_opus, seven_day_sonnet, overage), `OverageDisabledReason` (13 variants including out_of_credits, org_level_disabled, etc.)
- REQ-2: Early warning detection must implement tiered thresholds: 5-hour window at 0.9 utilization when 0.72 time elapsed, 7-day windows with multi-tier thresholds. The `compute_time_progress(resets_at, window_seconds)` function calculates time-relative warning position
- REQ-3: Rate limit messages must be subscription-aware: different messaging for Pro/Max vs Team/Enterprise vs Console users. Include upsell guidance, extra usage (overage) transitions, and special handling for ANT users (feedback channel '#briarpatch-cc')
- REQ-4: Mock rate limit infrastructure (ANT-only, gated by `USER_TYPE == "ant"`) must support 19 test scenarios: normal, session-limit-reached, overage-active, opus-limit, and 15+ more. Mock headers (20+ types) can be set individually or via scenario presets. Opus-specific mocks only fire when using Opus model
- REQ-5: Policy limits must enforce organization-level feature restrictions via HTTP polling: `is_policy_allowed(policy)` sync check (fail-open), initial load with 30-second timeout, 60-minute background refresh interval, ETag-based HTTP caching (304 Not Modified), file cache at `~/.claude/policy-limits.json`, retry with exponential backoff (max 5 retries). Eligible for Console users (API key) or OAuth Team/Enterprise only
- REQ-6: Rate limit error detection must identify rate limit errors via `is_rate_limit_error_message(text)` checking against 5 standard error prefixes
- REQ-7: The mock facade (`rateLimitMocking`) must isolate mock logic from production: `process_rate_limit_headers()` applies mocks if active, `check_mock_rate_limit_error()` generates synthetic 429 errors, `should_process_mock_limits()` gates on /mock-limits command state

## Acceptance Criteria

- [ ] AC-1: Rate limit headers are correctly parsed into ClaudeAILimits state — verified by header fixture tests
- [ ] AC-2: Early warnings fire at correct utilization/time thresholds — verified by 10+ threshold tests
- [ ] AC-3: All 19 mock scenarios produce correct mock headers — verified by scenario tests
- [ ] AC-4: Policy limits fail-open when endpoint unreachable — verified by timeout test
- [ ] AC-5: ETag caching prevents redundant policy fetches — verified by 304 response test
- [ ] AC-6: Subscription-aware messages differ for Pro vs Team vs Enterprise — verified by fixture tests

## Architecture

```
crates/claude-core/src/rate_limit/
├── mod.rs              (ClaudeAILimits state, header parsing)
├── early_warning.rs    (tiered threshold detection)
├── messages.rs         (subscription-aware message generation)
├── mock.rs             (19 scenarios, 20+ mock header types, ANT-only)
├── facade.rs           (rateLimitMocking isolation layer)
└── error.rs            (rate limit error detection)

crates/claude-config/src/policy/
├── mod.rs              (PolicyLimitsService: load, refresh, check)
├── fetch.rs            (HTTP polling with ETag, retry, file cache)
└── types.rs            (PolicyLimitsResponse, restriction schema)
```

## Open Questions

None.

## Out of Scope

- API retry logic for 429 responses (lives in `claude-core` query engine)
- UI display of rate limit warnings (lives in `claude-tui`)
