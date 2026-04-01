# Feature: Bridge & Remote Sessions (`claude-bridge` crate)

## Summary
Implement the bridge system that enables Claude Code to be controlled remotely from claude.ai/code and IDE extensions (VS Code, JetBrains). This covers the BridgeApiClient (8 methods for environment registration, work polling, heartbeat, session management), three transport types (HybridTransport for CCR v1, SSETransport for CCR v2, WebSocket for direct connections), JWT token refresh with expiry buffer, exponential backoff reconnection with 15-minute budget, and the remote session manager. The ~31 TypeScript files in `src/bridge/` plus 4 in `src/remote/` and 3 in `src/server/` total ~10K LOC.

## Requirements

- REQ-1: BridgeApiClient must implement all 8 methods from `src/bridge/types.ts`: `register_bridge_environment(config) → (environment_id, environment_secret)`, `poll_for_work(env_id, env_secret, signal?, reclaim_older_than_ms?)`, `acknowledge_work(env_id, work_id, session_token)`, `stop_work(env_id, work_id, force)`, `deregister_environment(env_id)`, `archive_session(session_id)`, `reconnect_session(env_id, session_id)`, `heartbeat_work(env_id, work_id, session_token) → {lease_extended, state}`
- REQ-2: CCR v1 transport must use HybridTransport: WebSocket for server-to-client messages (reads), HTTP POST to Session-Ingress for client-to-server messages (writes). CCR v2 transport must use SSE for reads plus CCRClient for writes via `/worker/*` endpoints, with epoch management, state reporting, and delivery tracking (`report_state()`, `report_metadata()`, `report_delivery()`, `flush()`)
- REQ-3: WebSocket reconnection must implement: initial delay 2,000ms, max delay 60,000ms (exponential backoff), ±25% jitter, reconnect budget of 15 minutes (`POLL_ERROR_GIVE_UP_MS = 15 * 60 * 1000`), max 5 reconnect attempts for WebSocket connections, sleep detection (gap > 60s resets budget), 30-second ping interval
- REQ-4: JWT token management must implement: decode payload without verification (`decode_jwt_payload`), extract expiry (`decode_jwt_expiry`), 5-minute refresh buffer (`TOKEN_REFRESH_BUFFER_MS = 5 * 60 * 1000`), fallback refresh interval of 30 minutes, max 3 consecutive refresh failures before giving up, 60-second retry delay between failures
- REQ-5: The `ReplBridgeTransport` trait must define: `write(message: StdoutMessage)`, `write_batch(messages: Vec<StdoutMessage>)`, `is_connected() → bool`, `set_on_data(callback)`, `set_on_close(callback)`. All three transport implementations (Hybrid, SSE, WebSocket) must implement this trait
- REQ-6: Remote session manager (`src/remote/RemoteSessionManager.ts`) must manage CCR session lifecycle: WebSocket client with auth headers, reconnect budget (~10 minutes), ping interval (30 seconds), and message format conversion between SDK and remote formats via the SDK message adapter
- REQ-7: The direct-connect session manager (`src/server/`) must handle local sessions where Claude Code runs as a server for IDE extensions, accepting connections and managing session state

## Acceptance Criteria

- [ ] AC-1: BridgeApiClient successfully registers an environment, polls for work, and acknowledges it — verified by integration test with mock CCR server
- [ ] AC-2: WebSocket reconnection respects exponential backoff with jitter — verified by testing 10 disconnect/reconnect cycles and checking delay distribution
- [ ] AC-3: JWT refresh triggers at exactly 5 minutes before expiry — verified by mock token with known expiry
- [ ] AC-4: Transport failover: if WebSocket fails, falls back gracefully — verified by killing the WebSocket connection mid-session
- [ ] AC-5: CCR v2 epoch tracking correctly manages message ordering — verified by out-of-order message test
- [ ] AC-6: Reconnection gives up after 15-minute budget expires — verified by time-advancing test
- [ ] AC-7: Sleep detection resets reconnect budget after 60-second gap — verified by simulating system sleep

## Architecture

```
crates/claude-bridge/
├── src/
│   ├── lib.rs
│   ├── api_client.rs       (BridgeApiClient: 8 methods, HTTP calls to CCR)
│   ├── transport/
│   │   ├── mod.rs          (ReplBridgeTransport trait)
│   │   ├── hybrid.rs       (CCR v1: WS reads + POST writes)
│   │   ├── sse.rs          (CCR v2: SSE reads + CCRClient writes)
│   │   └── websocket.rs    (Direct WebSocket transport)
│   ├── ccr/
│   │   ├── v1.rs           (CCR v1 protocol handling)
│   │   ├── v2.rs           (CCR v2: epochs, state, delivery)
│   │   └── client.rs       (CCRClient for /worker/* endpoints)
│   ├── jwt.rs              (JWT decode, expiry check, refresh scheduling)
│   ├── reconnect.rs        (exponential backoff, jitter, budget, sleep detection)
│   ├── remote/
│   │   ├── manager.rs      (RemoteSessionManager)
│   │   ├── websocket.rs    (SessionsWebSocket with auth + ping)
│   │   └── adapter.rs      (SDK message format adapter)
│   ├── server/
│   │   └── direct.rs       (direct-connect session manager)
│   └── error.rs            (BridgeError enum)
```

**Dependencies:**
- `tokio-tungstenite` (WebSocket client)
- `reqwest` (HTTP POST for writes, SSE for reads)
- `reqwest-eventsource` (SSE client for CCR v2)
- `base64` (JWT payload decoding)
- `serde_json` (JWT payload parsing)
- `claude-auth` (OAuth token access for auth headers)
- `claude-net` (proxy configuration, mTLS)

## Open Questions

None — the bridge protocol is well-defined by the TypeScript implementation.

## Out of Scope

- IDE extension code (VS Code, JetBrains — those are separate projects)
- OAuth token management (lives in `claude-auth`)
- Upstream proxy relay (lives in `claude-net`)
