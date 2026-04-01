# Feature: MCP Integration (`claude-mcp` crate)

## Summary
Implement the Model Context Protocol (MCP) client that connects to external tool/resource servers. This covers 6 transport types (Stdio, SSE, Streamable HTTP, WebSocket, plus IDE-specific variants), server lifecycle management (5 states: connected, failed, needs-auth, pending, disabled), tool schema translation (MCP tool definitions → native Tool trait wrappers), resource listing/reading, and cross-app authentication via ClaudeAuthProvider. The `src/services/mcp/` directory contains ~12 files with ~5K LOC.

## Requirements

- REQ-1: MCP client must support 6 transport types: StdioClientTransport (spawned process), SSEClientTransport (Server-Sent Events), StreamableHTTPClientTransport (Streamable HTTP spec), WebSocketTransport (custom implementation in `mcpWebSocketTransport.ts`), plus IDE-specific SSE and HTTP transports. Each transport must implement connect, disconnect, and message passing
- REQ-2: Server lifecycle must manage 5 states: `Connected` (tools/resources available), `Failed` (connection error, retry logic), `NeedsAuth` (OAuth/XAA authentication required), `Pending` (connection in progress), `Disabled` (user-disabled). State transitions must be tracked and reported to the UI
- REQ-3: Tool schema translation must convert MCP tool definitions to native `Tool` trait implementations via `McpToolWrapper`. The wrapper delegates `call()` to the MCP client's `call_tool()`, maps `input_schema` from MCP JSON Schema to the native schema format, and constructs tool names as `"{server_name}_{tool_name}"` via `build_mcp_tool_name()`
- REQ-4: The MCP client must implement: `list_tools() → Vec<McpToolDefinition>`, `call_tool(name, tool_use_id, args) → ToolResult`, `list_resources() → Vec<McpResource>`, `read_resource(uri) → ResourceContent`, `list_prompts() → Vec<McpPrompt>`
- REQ-5: Cross-app authentication via ClaudeAuthProvider must support XAA (cross-app access) tokens for MCP servers that require OAuth. The MCP proxy URL (`mcp-proxy.anthropic.com`) must be used with path template `/v1/mcp/{server_id}`
- REQ-6: Server configuration must be loaded from `.mcp.json` (project-level) and `~/.claude/settings.json` (user-level), with enable/disable controls per server. Enterprise policy can allowlist or denylist specific MCP servers
- REQ-7: MCP tool deferred loading must support the ToolSearch pattern: tools marked `shouldDefer` are not sent to the model until explicitly searched, reducing prompt size. `ToolSearchTool` searches deferred tools by name/description

## Acceptance Criteria

- [ ] AC-1: Stdio transport spawns a process, sends initialize, and receives tool list — verified by integration test with a mock MCP server binary
- [ ] AC-2: SSE transport connects to an HTTP endpoint and receives server-sent events — verified by mock SSE server
- [ ] AC-3: McpToolWrapper correctly delegates tool calls to the MCP client — verified by mock client test
- [ ] AC-4: Tool name construction follows `"{server}_{tool}"` format — verified by unit test
- [ ] AC-5: Server state transitions follow the 5-state lifecycle correctly — verified by state machine test
- [ ] AC-6: NeedsAuth state triggers OAuth flow and retries connection — verified by integration test
- [ ] AC-7: Deferred tools are excluded from the initial tool list and loaded on ToolSearch — verified by integration test

## Architecture

```
crates/claude-mcp/
├── src/
│   ├── lib.rs
│   ├── client.rs           (McpClient: list_tools, call_tool, list_resources, read_resource)
│   ├── transport/
│   │   ├── mod.rs          (McpTransport trait)
│   │   ├── stdio.rs        (StdioClientTransport: process spawn + stdin/stdout)
│   │   ├── sse.rs          (SSEClientTransport: EventSource)
│   │   ├── http.rs         (StreamableHTTPClientTransport)
│   │   └── websocket.rs    (WebSocketTransport)
│   ├── server/
│   │   ├── lifecycle.rs    (5-state management, health checks)
│   │   ├── registry.rs     (server discovery from .mcp.json + settings)
│   │   └── config.rs       (McpServerConfig parsing)
│   ├── tools/
│   │   ├── wrapper.rs      (McpToolWrapper: impl Tool for MCP tools)
│   │   ├── schema.rs       (JSON Schema translation)
│   │   └── deferred.rs     (deferred tool loading for ToolSearch)
│   ├── auth.rs             (ClaudeAuthProvider, XAA tokens)
│   └── error.rs            (McpError enum)
```

**Dependencies:**
- `tokio` (async process management for Stdio transport)
- `reqwest` + `reqwest-eventsource` (HTTP/SSE transports)
- `tokio-tungstenite` (WebSocket transport)
- `serde_json` (JSON-RPC message handling)
- `claude-types` (Tool trait for McpToolWrapper)
- `claude-auth` (OAuth tokens for authenticated servers)

## Open Questions

None.

## Out of Scope

- MCP server implementations (we only implement the client)
- MCP proxy server infrastructure (that's Anthropic's server)
- ToolSearchTool implementation (lives in `claude-tools`, uses the deferred loading API)
