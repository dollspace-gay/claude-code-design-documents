# Feature: Networking & Proxy (`claude-net` crate)

## Summary
Implement the networking infrastructure that handles HTTP client creation with proxy support, mTLS configuration, NO_PROXY pattern matching, and the upstream CONNECT-over-WebSocket relay for container environments (CCR). The ~800 LOC in `src/upstreamproxy/` implements a local TCP relay that tunnels HTTP CONNECT requests through a WebSocket connection for credential injection in managed environments.

## Requirements

- REQ-1: Proxy configuration must support: `HTTPS_PROXY`/`HTTP_PROXY` environment variables, `NO_PROXY` pattern matching (exact hostname, domain suffix with `.`, wildcard `*`, port-specific patterns like `host:port`), and the `should_bypass_proxy(host)` function implementing all 4 pattern types
- REQ-2: mTLS must support: client certificate + key from file paths (`CLAUDE_CODE_MTLS_CERT`, `CLAUDE_CODE_MTLS_KEY`), CA bundle override (`CLAUDE_CODE_CA_BUNDLE`), and both `rustls` (preferred) and `native-tls` (fallback) backends. The `create_http_client(config)` function builds a `reqwest::Client` with proxy and mTLS configured
- REQ-3: WebSocket client creation must apply the same proxy and TLS configuration: `create_ws_connector(config)` produces a `tokio_tungstenite::Connector` that routes through the configured proxy with mTLS when applicable
- REQ-4: The upstream proxy relay (`src/upstreamproxy/`) must implement a local TCP server that: (1) listens on a local port, (2) accepts HTTP CONNECT requests, (3) tunnels traffic through a WebSocket connection to the upstream proxy server, (4) injects session credentials into the tunnel handshake. The relay uses hand-encoded protobuf for tunnel chunks: `encode_chunk(data) → Vec<u8>`, `decode_chunk(buf) → Option<Vec<u8>>`
- REQ-5: The relay must return an `UpstreamProxyRelay` handle with the assigned port and a stop function for cleanup
- REQ-6: Preconnect optimization: initiate TCP/TLS connections to known API endpoints during startup (before the first API request) to reduce first-request latency

## Acceptance Criteria

- [ ] AC-1: HTTP client routes through proxy when HTTPS_PROXY is set — verified by mock proxy test
- [ ] AC-2: NO_PROXY bypasses proxy for matching hosts — verified by 10+ pattern/host test pairs
- [ ] AC-3: mTLS client certificate is sent with requests — verified by mock TLS server that requires client certs
- [ ] AC-4: Upstream relay tunnels TCP through WebSocket — verified by integration test connecting through the relay
- [ ] AC-5: Protobuf chunk encode/decode round-trips correctly — verified by property test
- [ ] AC-6: Preconnect reduces first-request latency by at least 100ms — verified by benchmark

## Architecture

```
crates/claude-net/
├── src/
│   ├── lib.rs
│   ├── proxy.rs            (ProxyConfig, NO_PROXY parsing, should_bypass_proxy)
│   ├── client.rs           (create_http_client, create_ws_connector)
│   ├── mtls.rs             (mTLS certificate loading, rustls/native-tls)
│   ├── upstream/
│   │   ├── relay.rs        (UpstreamProxyRelay: TCP listener + WS tunnel)
│   │   └── proto.rs        (hand-encoded protobuf: encode_chunk, decode_chunk)
│   ├── preconnect.rs       (parallel TCP/TLS preconnect to known endpoints)
│   └── error.rs            (NetError enum)
```

**Dependencies:**
- `reqwest` (HTTP client with proxy support)
- `tokio-tungstenite` (WebSocket for relay)
- `rustls` + `rustls-pemfile` (TLS with client certs)
- `native-tls` (platform TLS fallback)
- `tokio::net` (TCP listener for relay)
- `prost` (protobuf chunk encoding — or hand-encode for simplicity matching TS)

## Open Questions

None.

## Out of Scope

- API endpoint URLs and routing (lives in `claude-auth` and `claude-core`)
- OAuth token injection into requests (lives in `claude-auth`)
- Bridge WebSocket connections (lives in `claude-bridge`, uses client from `claude-net`)
