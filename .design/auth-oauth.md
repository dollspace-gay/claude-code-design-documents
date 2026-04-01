# Feature: Authentication & OAuth (`claude-auth` crate)

## Summary
Implement the complete authentication system for the Rust rewrite as the `claude-auth` crate. This covers OAuth 2.0 with PKCE (browser-based and manual paste flows), multi-provider support (FirstParty, Bedrock, Vertex, Foundry), secure token storage (platform keychain + file fallback), the 8-level auth priority chain, automatic token refresh with deduplication, and 401 error recovery. All OAuth endpoints, client IDs, scopes, and flow behavior must match the TypeScript implementation exactly to maintain session continuity.

## Requirements

- REQ-1: OAuth 2.0 Authorization Code flow with PKCE must implement the exact sequence: (1) generate 32-byte code verifier (base64url), (2) SHA256 hash to base64url code challenge, (3) open browser to authorize URL with S256 challenge method, (4) listen on ephemeral localhost port for redirect callback at `/callback`, (5) exchange code + verifier for tokens at TOKEN_URL with 15-second timeout. The PKCE crypto functions (`generate_code_verifier`, `generate_code_challenge`, `generate_state`) must produce output format-identical to TypeScript's `crypto.ts`
- REQ-2: Three OAuth environments must be supported with identical flow structure — Production (two authorize URLs: CONSOLE at `platform.claude.com/oauth/authorize` and CLAUDE_AI at `claude.com/cai/oauth/authorize`, client_id `9d1c250a-e61b-44d9-88ed-5944d1962f5e`, mcp-proxy.anthropic.com), Staging (staging.ant.dev endpoints), Local (localhost:8000/4000/3000). Environment selection via `get_oauth_config_type()` matching TypeScript's `getOAuthConfigType()`. Additionally, `CLAUDE_CODE_CUSTOM_OAUTH_URL` env var must be supported for allowlisted FedStart deployments
- REQ-3: Authentication operates as two parallel chains. **Bearer/OAuth token chain** (`get_auth_token_source()`, 7 levels, first match wins): (1) `--bare` mode: only apiKeyHelper from settings, (2) ANTHROPIC_AUTH_TOKEN env var (if not in managed OAuth context), (3) CLAUDE_CODE_OAUTH_TOKEN env var, (4) CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR from pipe FD, (5) CCR_OAUTH_TOKEN_FILE disk fallback, (6) apiKeyHelper external command (trust-approved), (7) keychain/secure storage OAuth tokens with Claude.ai scope, (8) none. **API key chain** (`get_anthropic_api_key_with_source()`, separate function): handles X-Api-Key header injection from ANTHROPIC_API_KEY env var, OAuth-derived keys, or apiKeyHelper. These two chains are independent — the bearer token chain does NOT check ANTHROPIC_API_KEY
- REQ-4: Token refresh must implement: 5-minute expiry buffer, in-flight promise deduplication for concurrent callers (keyed on retryCount=0 + force=false), MAX_RETRIES=5 with 1-2 second random jittered delay per retry (NOT exponential — flat `1000 + random()*1000` ms), and cross-process staleness detection via `.credentials.json` mtime checking. Note: `invalid_grant` detection exists only in provider-specific paths (Vertex/MCP), NOT in the core first-party OAuth refresh
- REQ-5: The 401 error handler must: (1) clear memoized cache, (2) read keychain async, (3) if keychain has different token than failed token → use it (another process refreshed), (4) if same token → force refresh bypassing local expiry check, (5) deduplicate concurrent 401 handlers by failed access token
- REQ-6: Multi-provider support must implement four `ApiProvider` variants: FirstParty (api.anthropic.com with OAuth/API key), Bedrock (AWS STS credentials via `aws-sdk-sts`), Vertex (Google service account via `gcp-auth`), Foundry (Azure DefaultCredential). Each provider needs its own SDK client initialization, credential refresh, and header requirements
- REQ-7: Secure storage must use platform-appropriate backends: macOS Keychain (via `security` CLI or `keyring` crate), Linux `libsecret`/`kwallet`/file fallback, Windows Credential Manager. The `keyring` crate handles this abstraction. Fallback to `~/.claude/.credentials.json` (mode 0600) when no system keychain is available
- REQ-8: OAuth scopes must match exactly: `CLAUDE_AI_OAUTH_SCOPES = ["user:profile", "user:inference", "user:sessions:claude_code", "user:mcp_servers", "user:file_upload"]`, `CONSOLE_OAUTH_SCOPES = ["org:create_api_key", "user:profile"]`, `ALL_OAUTH_SCOPES = dedupe(CONSOLE + CLAUDE_AI)` (used as default in `build_auth_url()`). The `OAUTH_BETA_HEADER = "oauth-2025-04-20"` is sent on API requests authenticated with OAuth tokens (via `anthropic-beta` header), NOT on token exchange/refresh requests to the OAuth token endpoint
- REQ-9: The AuthCodeListener must: bind to OS-assigned port (port 0), validate CSRF state parameter on callback, support both automatic browser redirect and manual paste fallback, send 302 redirect to appropriate success URL (CLAUDEAI_SUCCESS_URL if Claude AI scope, CONSOLE_SUCCESS_URL otherwise), handle pending responses before server close
- REQ-10: Profile fetching must call `GET {BASE_API_URL}/api/oauth/profile` with Bearer token (10s timeout) and map `organization.organization_type` to SubscriptionType (claude_max, claude_pro, claude_enterprise, claude_team), extract rate_limit_tier, billing_type, account/subscription timestamps, display_name

## Acceptance Criteria

- [ ] AC-1: PKCE flow produces valid code_verifier (43-128 chars, base64url), code_challenge (base64url SHA256), and state (base64url) — verified by property tests against RFC 7636
- [ ] AC-2: OAuth flow against a mock server completes successfully: browser opens URL, callback receives code, token exchange returns tokens — verified by integration test with local HTTP server
- [ ] AC-3: Auth priority chain returns the correct source for each of the 8 levels — verified by unit tests that set environment variables and mock keychain
- [ ] AC-4: Token refresh deduplicates concurrent callers — verified by spawning 10 concurrent refresh tasks and asserting exactly 1 HTTP request is made
- [ ] AC-5: 401 handler detects cross-process token update (different token in keychain) and avoids unnecessary refresh — verified by integration test
- [ ] AC-6: Secure storage reads/writes tokens on Linux, macOS, and Windows — verified by conditional compilation tests per platform
- [ ] AC-7: All three OAuth environments (production, staging, local) produce correctly-formed authorize URLs with proper client_id, scopes, redirect_uri, and PKCE parameters — verified by snapshot tests
- [ ] AC-8: The Bedrock provider creates valid STS credentials, the Vertex provider uses Google service account auth, and the Foundry provider uses Azure DefaultCredential — verified by mock-server integration tests per provider
- [ ] AC-9: Expired tokens trigger automatic refresh before API calls, not during — verified by checking the 5-minute buffer calculation
- [ ] AC-10: `OAuthTokens` struct serializes to JSON identically to the TypeScript version — verified by roundtrip test against TypeScript-generated fixtures

## Architecture

**File structure:**
```
crates/claude-auth/
├── src/
│   ├── lib.rs              (pub re-exports, AuthService facade)
│   ├── oauth/
│   │   ├── mod.rs          (OAuthService: startOAuthFlow, waitForAuthorizationCode)
│   │   ├── client.rs       (buildAuthUrl, exchangeCodeForTokens, refreshOAuthToken, fetchProfileInfo)
│   │   ├── crypto.rs       (generate_code_verifier, generate_code_challenge, generate_state)
│   │   ├── listener.rs     (AuthCodeListener: HTTP server on ephemeral port)
│   │   └── config.rs       (OAuthConfig for 3 environments + custom URL override)
│   ├── priority.rs         (get_auth_token_source: 8-level priority chain)
│   ├── tokens.rs           (OAuthTokens, token expiry check, refresh with dedup)
│   ├── provider.rs         (ApiProvider enum: FirstParty, Bedrock, Vertex, Foundry)
│   ├── storage.rs          (SecureStorage trait + keyring/file backends)
│   ├── profile.rs          (fetchProfileInfo, SubscriptionType, RateLimitTier)
│   └── error.rs            (AuthError enum via thiserror)
```

**Key implementation details from TypeScript source:**

The `OAuthService` in TypeScript (`src/services/oauth/index.ts`) creates a local HTTP server that listens for the browser redirect after user authorization. The server binds to port 0 (OS-assigned), and the port number is embedded in the redirect_uri sent to the OAuth provider. This must be preserved exactly — some corporate firewalls block specific port ranges.

Token refresh uses a deduplication pattern: a module-level `Option<Shared<Pin<Box<dyn Future>>>>` holds the in-flight refresh future. Concurrent callers await the same future instead of issuing parallel refresh requests. The `force` parameter bypasses the expiry check but still deduplicates.

The auth priority chain in `src/utils/auth.ts` (lines 152-206) uses early returns, so the Rust implementation should use a chain of `if let` / `match` expressions that short-circuit at the first successful source.

**Dependencies:**
- `reqwest` (HTTP client for token exchange, profile fetch)
- `keyring` (platform secure storage)
- `sha2` + `base64` (PKCE crypto)
- `tokio` (async runtime for HTTP server)
- `hyper` or `tiny_http` (lightweight HTTP server for AuthCodeListener)
- `oauth2` crate (optional — may use directly if it matches the custom flow, but TypeScript's implementation is custom, not library-based)
- `aws-config` + `aws-sdk-sts` (Bedrock provider)
- `gcp-auth` (Vertex provider)
- `azure_identity` (Foundry provider)

## Open Questions

### Q1: Browser opening mechanism
**Decision**: Use the `open` crate (`open::that(url)`) — it is the direct Rust equivalent of the `open` npm package used by TypeScript. It delegates to `xdg-open` on Linux, `open` on macOS, and `start` on Windows. If opening fails (headless/SSH), it falls back silently to the manual paste flow, matching current behavior.

## Out of Scope

- MCP proxy authentication (that uses OAuth tokens but lives in `claude-mcp`)
- API request signing / header injection (that lives in `claude-core`'s SDK client)
- Permission mode management (that lives in `claude-state` / `claude-types`)
- User onboarding flow UI (that lives in `claude-tui`)
