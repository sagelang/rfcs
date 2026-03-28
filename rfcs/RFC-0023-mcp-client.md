# RFC-0023: MCP Client Integration

| Field      | Value                                      |
|------------|--------------------------------------------|
| RFC        | 0023                                       |
| Title      | MCP Client Integration                     |
| Status     | Draft                                      |
| Category   | Language / Runtime                         |
| Version    | 2.2.0                                      |
| Depends on | RFC-0011 (Tool Support)                    |

---

## 1. Summary

This RFC adds Model Context Protocol (MCP) client support to Sage, allowing agents to consume tools provided by any MCP server. It introduces two complementary modes of integration:

1. **Typed MCP tools** — user-declared tool interfaces backed by MCP servers, with full compile-time type checking.
2. **Dynamic MCP** — a standard library module (`mcp`) for runtime tool discovery and invocation without compile-time declarations.

Both modes support the stdio and Streamable HTTP transports defined in the MCP specification (2025-11-25). Typed tools are the primary path for production use; dynamic MCP serves orchestration and exploration scenarios where the tool set is not known at compile time.

This RFC also adds `sage tools` CLI commands for inspecting MCP servers, generating tool declarations, and verifying signatures.

---

## 2. Motivation

### 2.1 The ecosystem is already there

MCP has grown to over 7,000 servers covering filesystems, GitHub, Slack, Postgres, Google Drive, Puppeteer, and hundreds of domain-specific APIs. Every one of those servers is currently inaccessible to Sage agents. A Sage program that needs to interact with GitHub must either shell out to `gh`, make raw HTTP calls and parse JSON by hand, or use an extern Rust function wrapping a GitHub client library.

With MCP client support, the same program writes:

```sage
agent IssueFiler {
    use GitHub

    on start {
        let issue = try GitHub.create_issue(
            "sagelang/sage",
            "Bug: divine timeout on large prompts",
            "When the prompt exceeds 4000 tokens..."
        );
        print("Filed: " ++ issue.url);
        yield(issue.number);
    }

    on error(e) {
        yield(-1);
    }
}
```

The MCP server handles authentication, pagination, rate limiting, and API versioning. The Sage program handles types and logic.

### 2.2 Typed tools are the right default

RFC-0011 §14 sketched a dynamic MCP path using `mcp.connect()` and `mcp.call()` with untyped `Map<String, String>` arguments. That is useful for orchestration, but it gives up everything that makes Sage's tool system valuable: compile-time checking, autocompletion, clear error messages, and documentation-as-code.

The primary integration path should preserve the typed tool model. A user declares a `tool` interface in Sage — exactly as they do for built-in tools — and configures the MCP server in `grove.toml`. The compiler checks every call. The runtime dispatches to the MCP server. The user gets the same experience as `use Http`, but backed by any MCP server in the ecosystem.

### 2.3 Sage agents map naturally to MCP clients

An MCP client manages a stateful connection to a server, negotiates capabilities, and dispatches tool calls. A Sage agent is a stateful unit that manages connections (tool clients), has a lifecycle (start/stop), and dispatches fallible operations. The mapping is structural, not forced:

| MCP concept | Sage concept |
|-------------|--------------|
| Client connection | Agent tool client field |
| `tools/call` | `ToolName.function(args)` expression |
| Tool result (JSON) | Typed return value (record/enum/primitive) |
| Tool error | `SageError::Tool` via `try`/`catch` |
| Server lifecycle | Agent lifecycle (`on start` / `on resting`) |
| Capability negotiation | Compile-time `use` declaration + runtime init |

---

## 3. Design Overview

### 3.1 Architecture

```
                    ┌─────────────────────────────────┐
                    │         Sage Program             │
                    │                                  │
                    │  agent A {                       │
                    │      use Http        (built-in)  │
                    │      use GitHub      (MCP)       │
                    │      use Slack       (MCP)       │
                    │  }                               │
                    └──────┬──────────┬────────────────┘
                           │          │
              ┌────────────┘          └────────────────┐
              ▼                                        ▼
    ┌─────────────────┐                    ┌───────────────────┐
    │  HttpClient      │                    │  McpToolClient    │
    │  (reqwest,       │                    │  (JSON-RPC 2.0)   │
    │   direct impl)   │                    │                   │
    └─────────────────┘                    └─────┬─────────────┘
                                                 │
                                      ┌──────────┴──────────┐
                                      ▼                     ▼
                               ┌────────────┐       ┌──────────────┐
                               │   stdio    │       │  Streamable  │
                               │ transport  │       │    HTTP      │
                               └─────┬──────┘       └──────┬───────┘
                                     │                     │
                                     ▼                     ▼
                               ┌────────────┐       ┌──────────────┐
                               │ MCP Server │       │  MCP Server  │
                               │ (subprocess│       │  (remote)    │
                               └────────────┘       └──────────────┘
```

Built-in tools (`Http`, `Database`, `Fs`, `Shell`) remain unchanged — they are direct Rust implementations for performance. MCP tools go through `McpToolClient`, which implements the MCP JSON-RPC protocol over the configured transport.

### 3.2 What changes

| Layer | Change |
|-------|--------|
| **Parser** | No changes — `tool` declarations and `use ToolName` already parse |
| **Checker** | Accept user-defined tools from `grove.toml` MCP config in addition to built-ins |
| **Codegen** | Generate `McpToolClient` fields for MCP-backed tools; generate JSON serialisation for arguments and deserialisation for results |
| **Runtime** | New `mcp` module: `McpTransport` trait, `StdioTransport`, `StreamableHttpTransport`, `McpToolClient`, connection pooling |
| **CLI** | New `sage tools` subcommand for inspecting, generating, and verifying MCP tools |
| **Stdlib** | New `mcp` module for dynamic MCP |

---

## 4. Typed MCP Tools

### 4.1 Tool declaration

Users declare MCP tool interfaces using the existing `tool` syntax from RFC-0011. This is unchanged — it already describes typed function signatures:

```sage
// src/tools/github.sg

record Issue {
    number: Int,
    url: String,
    title: String,
    state: String,
}

record PullRequest {
    number: Int,
    url: String,
    title: String,
    merged: Bool,
}

pub tool GitHub {
    fn create_issue(repo: String, title: String, body: String) -> Issue
    fn list_issues(repo: String, state: String) -> List<Issue>
    fn get_pull_request(repo: String, number: Int) -> PullRequest
    fn create_pull_request(repo: String, title: String, body: String, head: String, base: String) -> PullRequest
}
```

Every function in a `tool` declaration is implicitly fallible. Calls must use `try` or `catch`.

### 4.2 grove.toml configuration

The MCP server backing a tool is configured in `grove.toml`:

```toml
[project]
name = "my_app"
entry = "src/main.sg"

# stdio transport — launch server as subprocess
[tools.GitHub]
transport = "stdio"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
env = { GITHUB_TOKEN = "$GITHUB_TOKEN" }

# Streamable HTTP transport — connect to remote server
[tools.Slack]
transport = "http"
url = "https://mcp.slack.example.com/mcp"
auth = "bearer"
token_env = "SLACK_MCP_TOKEN"

# Alternative: OAuth 2.1 auth
[tools.CloudAPI]
transport = "http"
url = "https://api.cloud.example.com/mcp"
auth = "oauth"
client_id_env = "CLOUD_CLIENT_ID"
authorization_url = "https://auth.cloud.example.com/authorize"
token_url = "https://auth.cloud.example.com/token"
scopes = ["tools:read", "tools:write"]
```

**Configuration fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `transport` | Yes | `"stdio"` or `"http"` |
| `command` | stdio only | Executable to launch |
| `args` | No | Arguments for stdio command |
| `env` | No | Environment variables passed to stdio server. Values starting with `$` are resolved from the host environment |
| `url` | http only | MCP server endpoint URL |
| `auth` | No | `"bearer"` or `"oauth"`. Default: none |
| `token_env` | bearer only | Environment variable containing the bearer token |
| `client_id_env` | oauth only | Environment variable containing OAuth client ID |
| `authorization_url` | oauth only | OAuth authorization endpoint |
| `token_url` | oauth only | OAuth token endpoint |
| `scopes` | No | OAuth scopes to request |
| `timeout_ms` | No | Per-call timeout in milliseconds. Default: `30000` |
| `connect_timeout_ms` | No | Connection timeout. Default: `10000` |

The tool name in `[tools.GitHub]` must match a `tool GitHub { ... }` declaration in the project source (case-sensitive). The checker enforces this at compile time (see §8).

### 4.3 Agent usage

Agents use MCP tools identically to built-in tools:

```sage
mod tools::github;
use tools::github::{GitHub, Issue};

agent IssueFiler {
    use GitHub

    title: String
    body: String

    on start {
        let issue = try GitHub.create_issue("sagelang/sage", self.title, self.body);
        print("Filed #" ++ int_to_str(issue.number) ++ ": " ++ issue.url);
        yield(issue.number);
    }

    on error(e) {
        print("Failed to file issue: " ++ e.message);
        yield(-1);
    }
}

run IssueFiler { title: "Test issue", body: "This is a test" };
```

From the user's perspective, `GitHub.create_issue(...)` is indistinguishable from `Http.get(...)`. The compiler checks argument types and return types. The runtime dispatches to the configured MCP server.

### 4.4 Argument and result mapping

Sage types are mapped to MCP JSON Schema types for the `tools/call` request and response:

| Sage type | JSON Schema / JSON value |
|-----------|--------------------------|
| `Int` | `integer` / JSON number |
| `Float` | `number` / JSON number |
| `Bool` | `boolean` / JSON `true`/`false` |
| `String` | `string` / JSON string |
| `Unit` | — (omitted) |
| `List<T>` | `array` / JSON array |
| `Map<String, V>` | `object` / JSON object |
| `Option<T>` | nullable `T` / JSON value or `null` |
| `(A, B, C)` | `array` / JSON array (positional) |
| `record Foo { x: Int, y: String }` | `object` / `{"x": 1, "y": "hello"}` |
| `enum Status { Active, Inactive }` | `string` / `"Active"` or `"Inactive"` |
| `enum Result { Ok(Int), Err(String) }` | tagged union / `{"Ok": 42}` or `{"Err": "msg"}` |

**Arguments** are serialised as a JSON object where parameter names are keys:
```json
{
    "repo": "sagelang/sage",
    "title": "Bug report",
    "body": "Description here"
}
```

**Results** are deserialised from the MCP `tools/call` response. The response `content` array is processed as follows:
1. If the response contains `structuredContent` (JSON matching `outputSchema`), deserialise directly into the declared return type.
2. If the response contains a single `text` content item, attempt JSON deserialisation into the declared return type. If the return type is `String`, use the text value directly.
3. If deserialisation fails, return `SageError::Tool` with details.

### 4.5 MCP function name mapping

Sage tool function names map to MCP tool names. The MCP server may use different naming conventions (e.g., `create_issue` vs `create-issue` vs `createIssue`). By default, Sage uses the function name as-is. An explicit mapping can be provided:

```sage
pub tool GitHub {
    #[mcp_name = "create-issue"]
    fn create_issue(repo: String, title: String, body: String) -> Issue
}
```

The `#[mcp_name]` attribute overrides the MCP tool name used in the `tools/call` request. This is the only attribute introduced by this RFC.

---

## 5. Dynamic MCP

For scenarios where the tool set is not known at compile time — orchestration agents, tool exploration, meta-programming — a standard library module provides untyped MCP access:

```sage
agent ToolExplorer {
    server_url: String

    on start {
        // Connect to an MCP server
        let conn = try mcp_connect("http", self.server_url);

        // Discover available tools
        let tools = try mcp_list_tools(conn);
        for tool in tools {
            print("Tool: " ++ tool.name ++ " — " ++ tool.description);
        }

        // Call a tool dynamically
        let args = {"repo": "sagelang/sage", "state": "open"};
        let result = try mcp_call(conn, "list-issues", args);
        print("Result: " ++ result);

        // Disconnect
        try mcp_disconnect(conn);

        yield(0);
    }

    on error(e) {
        print("MCP error: " ++ e.message);
        yield(1);
    }
}
```

### 5.1 Stdlib functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `mcp_connect` | `(transport: String, target: String) -> McpConnection fails` | Connect to an MCP server. `transport` is `"stdio"` or `"http"`. `target` is the command (stdio) or URL (http). |
| `mcp_connect_with` | `(transport: String, target: String, config: Map<String, String>) -> McpConnection fails` | Connect with additional config (env vars for stdio, auth headers for http). |
| `mcp_list_tools` | `(conn: McpConnection) -> List<McpTool> fails` | List tools available on the server. |
| `mcp_call` | `(conn: McpConnection, tool: String, args: Map<String, String>) -> String fails` | Call a tool. Arguments and results are JSON strings. |
| `mcp_call_json` | `(conn: McpConnection, tool: String, args: String) -> String fails` | Call a tool with raw JSON argument string. Returns raw JSON result. |
| `mcp_disconnect` | `(conn: McpConnection) -> Unit fails` | Disconnect from the server. |
| `mcp_server_info` | `(conn: McpConnection) -> McpServerInfo fails` | Get server name, version, and capabilities. |

### 5.2 Types

```sage
// Opaque handle — cannot be constructed in user code
record McpConnection {
    id: Int,
}

record McpTool {
    name: String,
    description: String,
    input_schema: String,   // JSON Schema as string
}

record McpServerInfo {
    name: String,
    version: String,
}
```

`McpConnection` is an opaque handle backed by an integer ID. The runtime maintains a connection registry keyed by ID. Connections are closed when the owning agent terminates if not explicitly disconnected.

### 5.3 Trade-offs

Dynamic MCP sacrifices compile-time safety for flexibility:
- Arguments are `Map<String, String>` — no type checking on keys or values
- Results are raw JSON strings — the caller must parse them
- Tool names are strings — typos are runtime errors

This is appropriate for orchestration, exploration, and meta-agents. Production code should prefer typed tool declarations.

---

## 6. Transport Implementation

### 6.1 stdio

The Sage runtime launches the MCP server as a subprocess, communicating over stdin/stdout with newline-delimited JSON-RPC 2.0 messages.

**Lifecycle:**
1. Runtime spawns the subprocess using `command` and `args` from `grove.toml`
2. Environment variables from `env` are injected (with `$VAR` references resolved)
3. Runtime sends `initialize` request with Sage's client info and capabilities
4. Server responds with its capabilities and server info
5. Runtime sends `notifications/initialized`
6. Normal operation: `tools/list`, `tools/call` requests
7. On agent shutdown or program exit: close stdin, wait for process exit, SIGTERM after timeout

**Process management:**
- One subprocess per MCP server configuration, shared across all agents using that tool
- Subprocess is started lazily on first tool call (not at program startup)
- If the subprocess exits unexpectedly, subsequent tool calls return `SageError::Tool` with a clear message
- stderr from the subprocess is forwarded to Sage's trace system as `mcp.server.log` events

### 6.2 Streamable HTTP

The Sage runtime connects to a remote MCP server over HTTP, following the Streamable HTTP transport specification.

**Lifecycle:**
1. Runtime sends `initialize` as HTTP POST to the configured URL
2. Request includes `Accept: application/json, text/event-stream` and `MCP-Protocol-Version: 2025-11-25`
3. Server responds with capabilities and a `MCP-Session-Id` header
4. Runtime stores session ID and includes it on all subsequent requests
5. Normal operation: POST requests for `tools/list`, `tools/call`
6. On shutdown: HTTP DELETE with session ID to terminate the session

**Connection details:**
- Uses `reqwest` (already a dependency) for HTTP requests
- Supports both `application/json` (simple response) and `text/event-stream` (SSE for streaming/notifications)
- Connection pooling is handled by `reqwest`'s built-in connection pool
- Timeout configuration per `grove.toml` settings

### 6.3 Protocol version negotiation

The Sage MCP client advertises protocol version `2025-11-25`. If the server responds with a different version, the client checks compatibility:
- Same version: proceed
- Older version the client can support: proceed with degraded capabilities
- Incompatible version: return `SageError::Tool` with message "MCP server requires protocol version X, Sage supports Y"

---

## 7. Authentication

### 7.1 stdio servers

stdio servers receive credentials through environment variables configured in `grove.toml`:

```toml
[tools.GitHub]
transport = "stdio"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
env = { GITHUB_TOKEN = "$GITHUB_TOKEN" }
```

The `$GITHUB_TOKEN` reference is resolved from the host environment at subprocess launch time. If the variable is not set, the runtime emits a clear startup error.

### 7.2 Bearer token (HTTP)

For simple token-based auth:

```toml
[tools.Slack]
transport = "http"
url = "https://mcp.slack.example.com/mcp"
auth = "bearer"
token_env = "SLACK_MCP_TOKEN"
```

The runtime includes `Authorization: Bearer <token>` on every HTTP request to this server.

### 7.3 OAuth 2.1 (HTTP)

For servers requiring OAuth, the runtime implements the OAuth 2.1 + PKCE flow specified in the MCP authorization specification:

```toml
[tools.CloudAPI]
transport = "http"
url = "https://api.cloud.example.com/mcp"
auth = "oauth"
client_id_env = "CLOUD_CLIENT_ID"
authorization_url = "https://auth.cloud.example.com/authorize"
token_url = "https://auth.cloud.example.com/token"
scopes = ["tools:read", "tools:write"]
```

**Flow:**
1. Runtime checks for cached token in `~/.sage/tokens/<tool_name>.json`
2. If no valid token, initiates authorization code flow with PKCE (S256)
3. Opens system browser for user authorization (interactive) or uses cached refresh token
4. Exchanges authorization code for access token
5. Caches token (with refresh token) for subsequent runs
6. Includes `Authorization: Bearer <access_token>` on all requests
7. On 401 response, attempts token refresh before failing

**Non-interactive mode:** If `SAGE_MCP_NONINTERACTIVE=1` is set, the OAuth flow skips browser-based authorization and fails with a clear error if no cached token is available. This is appropriate for CI/CD environments where tokens should be pre-provisioned.

---

## 8. Error Handling

### 8.1 Error categories

MCP tool calls can fail in several ways, all surfaced as `SageError::Tool`:

| Error | Cause | User-facing message |
|-------|-------|---------------------|
| Connection failure | Server unreachable, subprocess crash | `"MCP server 'GitHub' is not reachable: {details}"` |
| Protocol error | Invalid JSON-RPC response, version mismatch | `"MCP protocol error from 'GitHub': {details}"` |
| Tool not found | Server doesn't expose the called tool | `"MCP server 'GitHub' does not provide tool 'create_issue'"` |
| Tool execution error | Server returns `isError: true` | `"MCP tool 'create_issue' failed: {server_message}"` |
| Deserialisation error | Result JSON doesn't match declared return type | `"Cannot deserialise 'GitHub.create_issue' result into Issue: {details}"` |
| Authentication error | Token expired, OAuth flow failed | `"Authentication failed for MCP server 'GitHub': {details}"` |
| Timeout | Call exceeded configured timeout | `"MCP tool 'create_issue' timed out after 30000ms"` |

All errors flow through the standard `try`/`catch`/`on error` mechanisms. No new error kinds are introduced — `ErrorKind::Tool` (from RFC-0007) covers all MCP tool errors.

### 8.2 Compile-time error codes

| Code | Name | Condition |
|------|------|-----------|
| E080 | `McpToolNotConfigured` | Agent uses a `tool` that has no `[tools.X]` section in `grove.toml` and is not a built-in |
| E081 | `McpToolConfigInvalid` | `grove.toml` `[tools.X]` section has invalid or missing fields |
| E082 | `McpToolSignatureMismatch` | (with `--verify-tools`) Declared signature doesn't match MCP server manifest |
| E083 | `McpNameAttributeInvalid` | `#[mcp_name]` attribute value is not a string literal |

### 8.3 Connection resilience

- **Lazy connection:** MCP server connections are established on first tool call, not at program startup. An agent that never calls its MCP tools incurs no connection overhead.
- **No automatic retry:** Failed tool calls are not retried by the runtime. The Sage program controls retry logic via `retry(n) { ... }` blocks or explicit loops.
- **Subprocess restart:** If a stdio server subprocess exits, subsequent calls fail. The runtime does not auto-restart subprocesses — supervision trees should handle restart logic at the agent level.

---

## 9. Schema Verification

### 9.1 Build-time verification

`sage check --verify-tools` connects to each configured MCP server and verifies that declared tool signatures are compatible with the server's manifest:

```bash
$ sage check --verify-tools
Checking src/main.sg...
Connecting to MCP server 'GitHub' (stdio: npx -y @modelcontextprotocol/server-github)...
  ✓ create_issue — signature matches
  ✓ list_issues — signature matches
  ✗ get_pull_request — server expects parameter 'pull_number: integer', declaration has 'number: Int'
Error E082: Tool signature mismatch for GitHub.get_pull_request
```

This is opt-in because it requires network access (or subprocess launch) and the MCP server being available. Regular `sage check` validates the Sage source code without contacting servers.

### 9.2 CLI tool generation

`sage tools generate` connects to an MCP server and generates Sage tool declarations from its manifest:

```bash
$ sage tools generate --stdio "npx -y @modelcontextprotocol/server-github"
Connecting to MCP server...
Found 12 tools. Generating src/tools/github.sg...

$ cat src/tools/github.sg
// Auto-generated from MCP server: github-mcp-server v1.2.0
// Generated: 2026-03-28

record CreateIssueResult {
    number: Int,
    url: String,
    title: String,
    state: String,
}

pub tool GitHubServer {
    #[mcp_name = "create-issue"]
    fn create_issue(repo: String, title: String, body: String) -> CreateIssueResult

    #[mcp_name = "list-issues"]
    fn list_issues(repo: String, state: String) -> List<CreateIssueResult>

    // ... remaining tools ...
}
```

The generated file is a starting point. Users should review it, rename types and tools to taste, and remove functions they don't need. The generated file is checked into version control — it is source code, not a build artifact.

### 9.3 CLI commands

| Command | Description |
|---------|-------------|
| `sage tools list` | List all configured MCP tools from `grove.toml` |
| `sage tools inspect --stdio "command"` | Connect to a server, print its tool manifest |
| `sage tools inspect --http "url"` | Same, for HTTP servers |
| `sage tools generate --stdio "command" [-o file.sg]` | Generate Sage declarations from server manifest |
| `sage tools generate --http "url" [-o file.sg]` | Same, for HTTP servers |
| `sage tools verify` | Alias for `sage check --verify-tools` |

---

## 10. Testing & Mocking

MCP tools integrate with the existing test mock system from RFC-0012:

```sage
test "issue filing works" {
    mock tool GitHub.create_issue -> Issue {
        number: 42,
        url: "https://github.com/sagelang/sage/issues/42",
        title: "Test issue",
        state: "open",
    };

    let agent = summon IssueFiler { title: "Test", body: "Body" };
    let result = try await agent;
    assert_eq(result, 42);
}

test "handles server failure" {
    mock tool GitHub.create_issue -> fail("Server unavailable");

    let agent = summon IssueFiler { title: "Test", body: "Body" };
    let result = try await agent;
    assert_eq(result, -1);
}
```

**Mock behaviour:**
- `mock tool` intercepts before the MCP transport layer — no MCP server is contacted
- Works identically to built-in tool mocking
- Mock values must match the declared return type (enforced by the checker)
- Multiple mocks are consumed in FIFO order
- Dynamic MCP calls (`mcp_call`) can also be mocked via `mock tool mcp.call -> "json result"`

---

## 11. Implementation Plan

### Phase 1 — Runtime: MCP Transport (4–5 days)

New crate: `sage-mcp` (or module in `sage-runtime`).

- Implement `McpTransport` trait:
  ```rust
  #[async_trait]
  pub trait McpTransport: Send + Sync {
      async fn send(&self, request: JsonRpcRequest) -> Result<JsonRpcResponse, McpError>;
      async fn close(&self) -> Result<(), McpError>;
  }
  ```
- Implement `StdioTransport`:
  - Spawn subprocess with `tokio::process::Command`
  - Read/write newline-delimited JSON-RPC on stdin/stdout
  - Forward stderr to trace system
  - Handle process lifecycle (spawn, monitor, shutdown)
- Implement `StreamableHttpTransport`:
  - HTTP POST via `reqwest` for requests
  - Handle `application/json` and `text/event-stream` responses
  - Manage `MCP-Session-Id` header
  - Implement bearer token and OAuth 2.1 + PKCE auth
- Implement `McpClient`:
  - `initialize()` — handshake and capability negotiation
  - `list_tools()` — discover server tools
  - `call_tool(name, args_json)` — invoke a tool
  - `shutdown()` — graceful disconnect
- Connection pool: `McpConnectionPool` — one connection per server config, lazy init, shared across agents via `Arc`

**Dependencies:** `tokio`, `reqwest`, `serde_json`, `base64` (for PKCE). No new heavy dependencies.

### Phase 2 — Checker: User-Defined Tool Validation (2–3 days)

- Extend `Scope` to load tool declarations from user source files (not just `with_builtins()`)
- Load `grove.toml` `[tools.*]` sections during checking
- For each user-defined `tool` declaration:
  - Verify a matching `[tools.X]` section exists in `grove.toml` (E080)
  - Validate config fields (E081)
- Existing tool call checking (E036, E038, E039) works unchanged — the checker already validates against `ToolInfo` regardless of source
- Add E082, E083 error codes
- Add `#[mcp_name]` attribute parsing (single string literal only)

### Phase 3 — Codegen: MCP Tool Client Generation (3–4 days)

- For agents using MCP-backed tools, generate `McpToolClient` fields instead of built-in client fields:
  ```rust
  struct IssueFiler {
      title: String,
      body: String,
      github: Arc<McpToolClient>,  // injected from connection pool
  }
  ```
- Generate tool call dispatch:
  ```rust
  // GitHub.create_issue(repo, title, body) becomes:
  let __args = serde_json::json!({
      "repo": repo.clone(),
      "title": title.clone(),
      "body": body.clone(),
  });
  let __result = self.github.call_tool("create_issue", __args).await
      .map_err(|e| SageError::tool(e))?;
  let __typed: Issue = serde_json::from_value(__result)
      .map_err(|e| SageError::tool(format!("Cannot deserialise result: {}", e)))?;
  ```
- Handle `#[mcp_name]` attribute: use attribute value instead of function name in `call_tool()`
- Generate JSON serialisation for all argument types and deserialisation for return types
- Integrate with mock system: check `try_get_mock("GitHub", "create_issue")` before MCP dispatch

### Phase 4 — Stdlib: Dynamic MCP (2–3 days)

- Implement `mcp_connect`, `mcp_list_tools`, `mcp_call`, `mcp_call_json`, `mcp_disconnect`, `mcp_server_info` as runtime builtins
- Add `McpConnection`, `McpTool`, `McpServerInfo` to the type system
- Connection registry: `Arc<Mutex<HashMap<i64, McpClient>>>` keyed by connection ID
- Auto-cleanup: close connections for terminated agents

### Phase 5 — CLI: sage tools (2–3 days)

- `sage tools list` — parse `grove.toml`, print configured tools
- `sage tools inspect` — connect to server, print manifest as table
- `sage tools generate` — connect to server, generate `.sg` file with tool/record declarations
- `sage tools verify` — run checker with `--verify-tools` flag
- JSON Schema → Sage type mapping for `generate`:
  - `string` → `String`, `integer` → `Int`, `number` → `Float`, `boolean` → `Bool`
  - `object` with properties → `record`
  - `array` with items → `List<T>`
  - `oneOf` / enum → `enum` with variants

### Phase 6 — Tests & Documentation (2–3 days)

- Unit tests for MCP transport (mock TCP/subprocess)
- Integration tests with a test MCP server (bundled, minimal)
- Checker tests for E080-E083
- Codegen snapshot tests for MCP tool calls
- End-to-end test: stdio MCP server → Sage agent → tool call → result
- sage-book documentation:
  - New chapter: "MCP Tools" under Tools section
  - Update "Built-in Tools" overview to mention MCP tools
  - Add "Connecting to MCP Servers" guide
  - Add "sage tools" CLI reference

**Estimated total: 15–21 days**

---

## 12. Migration & Compatibility

### 12.1 No breaking changes

All existing Sage programs continue to work unchanged. Built-in tools remain built-in — their implementations are not moved to MCP. The `[tools.*]` configuration in `grove.toml` is additive.

### 12.2 Built-in tools stay native

`Http`, `Database`, `Fs`, and `Shell` remain direct Rust implementations. They are not proxied through MCP. This is the right trade-off:
- Zero protocol overhead for the most common tools
- No dependency on external MCP servers for basic operations
- WASM target compatibility (stdio/HTTP MCP transports don't work in browsers)

A user who wants the MCP version of a filesystem server (for sandboxing, auditing, etc.) can declare their own `tool McpFs { ... }` pointing to an MCP filesystem server — both can coexist.

### 12.3 Version bump

This feature ships as Sage **v2.2.0**. The minor version bump reflects new functionality with no breaking changes.

---

## 13. Open Questions

### 13.1 Resource and prompt support

MCP defines three primitives: **tools**, **resources**, and **prompts**. This RFC covers tools only. Resources (read-only data sources) and prompts (template-based prompt generation) are valuable but distinct features:

- **Resources** could map to a Sage `resource` declaration or be accessed via dynamic MCP functions (`mcp_list_resources`, `mcp_read_resource`).
- **Prompts** could integrate with `divine` — an MCP prompt could provide the system prompt or context for a `divine` call.

Both are deferred to a follow-up RFC. The transport and connection infrastructure from this RFC is fully reusable.

### 13.2 Streaming tool results

Some MCP tools return streaming responses. Sage's current model — a tool call returns a value — cannot represent streaming. A future `Stream<T>` type would be needed. This RFC's `McpTransport` supports streaming responses at the protocol level (SSE), but the Sage-facing API buffers the full result before returning.

### 13.3 Task support

MCP's experimental `tasks` primitive enables long-running operations with status polling. This could map naturally to Sage's agent model — an MCP task becomes an agent handle that can be `await`ed. Deferred until the tasks primitive stabilises in the MCP spec.

### 13.4 Sampling (reverse LLM calls)

MCP servers can request LLM completions from the client via `sampling/createMessage`. In Sage, this would mean the MCP server asking the Sage runtime to perform a `divine` call. This is a powerful composition pattern but raises trust and cost concerns. Deferred.

### 13.5 Tool call batching

MCP supports batch JSON-RPC requests. This RFC dispatches tool calls individually. A future optimisation could detect concurrent `summon`/`await` patterns and batch the underlying MCP calls. This is a runtime optimisation invisible to the user.

### 13.6 Connection sharing across agents

This RFC uses one MCP connection per server config, shared across agents. If two agents both `use GitHub`, they share the same `McpClient` instance. This is efficient but means a slow tool call from one agent could block another. The runtime uses `tokio` async internally, so this is only an issue if the MCP server itself is single-threaded. Connection-per-agent is an alternative but wastes resources for stdio servers.

---

## 14. Future Work

This RFC establishes the MCP client infrastructure. Subsequent work includes:

| Feature | Description | Depends on |
|---------|-------------|------------|
| **MCP Resources** | Access server-provided data sources | This RFC (transport layer) |
| **MCP Prompts** | Server-provided prompt templates for `divine` | This RFC + divine integration |
| **Sage as MCP Server** | Expose agents as MCP tools via `sage serve` | This RFC (protocol impl) + new RFC |
| **Tool call batching** | Batch concurrent MCP calls automatically | This RFC (runtime optimisation) |
| **Streaming results** | `Stream<T>` type for streaming MCP responses | New type system RFC |
| **MCP Sampling** | Allow MCP servers to request `divine` calls | This RFC + trust model RFC |
| **Grove registry MCP** | Package registry distributing tool declarations | RFC-0020 (suspended) + this RFC |

---

## 15. Example: Full Program with MCP Tools

A complete example demonstrating typed MCP tools, error handling, and multi-agent coordination:

**grove.toml:**
```toml
[project]
name = "issue_triage"
entry = "src/main.sg"

[tools.GitHub]
transport = "stdio"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
env = { GITHUB_TOKEN = "$GITHUB_TOKEN" }
```

**src/tools/github.sg:**
```sage
record Issue {
    number: Int,
    url: String,
    title: String,
    body: String,
    state: String,
}

record Label {
    name: String,
}

pub tool GitHub {
    #[mcp_name = "list-issues"]
    fn list_issues(repo: String, state: String) -> List<Issue>

    #[mcp_name = "add-labels"]
    fn add_labels(repo: String, issue_number: Int, labels: List<String>) -> List<Label>
}
```

**src/main.sg:**
```sage
mod tools::github;
use tools::github::{GitHub, Issue};

agent Classifier {
    issue: Issue

    on start {
        let labels = try divine(
            "Classify this GitHub issue into one or more labels.
Return ONLY a JSON array of label strings.
Valid labels: bug, feature, docs, performance, security, question

Title: {self.issue.title}
Body: {self.issue.body}"
        );
        yield(labels);
    }

    on error(e) {
        yield(["triage-needed"]);
    }
}

agent Triager {
    use GitHub

    repo: String

    on start {
        let issues = try GitHub.list_issues(self.repo, "open");
        print("Found " ++ int_to_str(len(issues)) ++ " open issues.");

        for issue in issues {
            // Classify each issue with the LLM
            let classifier = summon Classifier { issue: issue };
            let labels: List<String> = try await classifier;

            // Apply labels via MCP
            try GitHub.add_labels(self.repo, issue.number, labels);
            print("#" ++ int_to_str(issue.number) ++ " → " ++ join(labels, ", "));
        }

        yield(0);
    }

    on error(e) {
        print("Triage failed: " ++ e.message);
        yield(1);
    }
}

run Triager { repo: "sagelang/sage" };
```

This program:
1. Connects to the GitHub MCP server (launched as a subprocess)
2. Lists open issues (typed, compile-time checked)
3. Classifies each issue using `divine` (LLM integration)
4. Applies labels back via the MCP server
5. All tool calls are fallible with proper error handling

---

*Ward has reviewed this RFC and found no type errors.*
