# External Integrations

**Analysis Date:** 2026-04-14

## LLM Provider APIs

### Anthropic API (primary)
- Client: `api/src/providers/anthropic.rs` (`AnthropicClient`)
- Endpoint: `POST {base_url}/v1/messages` (streaming and non-streaming)
- Default base URL: `https://api.anthropic.com`
- Override: `ANTHROPIC_BASE_URL` env var
- Auth: `x-api-key` header for API keys, `Authorization: Bearer` for OAuth tokens
- Supports both API key + bearer token simultaneously (`ApiKeyAndBearer` variant)
- Additional endpoints:
  - `POST {base_url}/v1/messages/count_tokens` -- Context window preflight check
  - `POST {oauth_config.token_url}` -- OAuth token exchange and refresh
- Prompt caching: Supported via `prompt-caching-scope-2026-01-05` beta header
- Retry: Exponential backoff with jitter (1s-128s, max 8 retries)
- Retryable statuses: 408, 409, 429, 500, 502, 503, 504

### OpenAI-Compatible API
- Client: `api/src/providers/openai_compat.rs` (`OpenAiCompatClient`)
- Endpoint: `POST {base_url}/chat/completions`
- Wire format translation: Anthropic-shaped requests are converted to OpenAI `ChatCompletion` format
- Used by three provider configurations:
  - **OpenAI**: `OPENAI_API_KEY`, default `https://api.openai.com/v1`
  - **xAI (Grok)**: `XAI_API_KEY`, default `https://api.x.ai/v1`
  - **Alibaba DashScope (Qwen)**: `DASHSCOPE_API_KEY`, default `https://dashscope.aliyuncs.com/compatible-mode/v1`
- Supports provider-namespaced model routing (e.g., `openai/gpt-4.1-mini`, `qwen/qwen-max`)
- Streaming usage opt-in via `stream_options.include_usage` (OpenAI only)
- Tool message sanitization: Orphaned `role:tool` messages are dropped before sending
- Reasoning model detection: Strips tuning params for `o1/o3/o4/grok-3-mini/qwq/thinking` models
- gpt-5 series uses `max_completion_tokens` instead of `max_tokens`

### Provider Detection and Routing
- Logic: `api/src/providers/mod.rs`
- Resolution order: model name prefix > `OPENAI_BASE_URL` presence > env var auth sniffing > Anthropic fallback
- Model aliases: `opus` -> `claude-opus-4-6`, `sonnet` -> `claude-sonnet-4-6`, `haiku` -> `claude-haiku-4-5-20251213`, `grok` -> `grok-3`
- Foreign credential hints: When Anthropic auth fails but other provider keys are present, a hint is appended suggesting the correct routing prefix

### HTTP Client
- Library: `reqwest 0.12` with `rustls-tls` (no OpenSSL dependency)
- Client factory: `api/src/http_client.rs` (`build_http_client`, `build_http_client_with`)
- Proxy support: Honors `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` (case-insensitive)
- Config-file proxy: `proxy_url` setting for unified proxy
- No-proxy filtering: Supports domain/CIDR patterns via `reqwest::NoProxy`

## Authentication

### API Key Authentication
- Anthropic: `x-api-key` header from `ANTHROPIC_API_KEY`
- OpenAI/compatible: `Authorization: Bearer` from provider-specific env var
- Credential discovery falls back to `.env` file in CWD when env vars are absent
- Empty string values are treated as "not set"

### OAuth 2.0 (Anthropic)
- Implementation: `api/src/providers/anthropic.rs` (token exchange/refresh), `runtime/src/oauth.rs` (PKCE helpers)
- PKCE flow with S256 challenge method (`sha2` hashing)
- Token persistence: `$CLAW_CONFIG_HOME/oauth_credentials.json`
- Token refresh: Automatic when `expires_at` is in the past (if refresh token available)
- Local HTTP callback server on port 4545 (configurable via `OAuthConfig.callback_port`)
- Manual redirect URL support for headless environments
- Scopes: Configurable per OAuth config

## Model Context Protocol (MCP)

### MCP Client
- Implementation: `runtime/src/mcp_client.rs`, `runtime/src/mcp_stdio.rs`
- Transports supported:
  - **Stdio**: Spawns subprocess, communicates via stdin/stdout with LSP-style newline-delimited JSON-RPC framing
  - **SSE**: Server-Sent Events over HTTP
  - **HTTP**: Direct HTTP POST to MCP endpoint
  - **WebSocket**: WebSocket transport
  - **SDK**: In-process SDK transport
  - **Managed Proxy**: Remote proxy with URL + ID
- Protocol version: `2025-03-26`
- Lifecycle: `initialize` -> `tools/list` -> `tools/call` (with configurable timeouts)
- Initialize timeout: 10s (production), 200ms (test)
- List tools timeout: 30s (production), 300ms (test)
- Default tool call timeout: 60s (`DEFAULT_MCP_TOOL_CALL_TIMEOUT_MS`)
- Tool name prefix: `mcp__{server_name}__{tool_name}` (names are normalized)
- OAuth support for remote MCP servers (`McpOAuthConfig`)
- Lifecycle hardening: `runtime/src/mcp_lifecycle_hardened.rs` tracks phases, degradations, and failures

### MCP Server
- Implementation: `runtime/src/mcp_server.rs`
- Runs as a stdio-based JSON-RPC server (can be driven by Claude Desktop or `McpServerManager`)
- Exposes `initialize`, `tools/list`, and `tools/call` methods
- Tool handler is a `Box<dyn Fn(&str, &JsonValue) -> Result<String, String>>` callback
- Protocol version: `2025-03-26`

### MCP Tool Bridge
- Implementation: `runtime/src/mcp_tool_bridge.rs`
- Stateful client registry connecting tool handlers to MCP server connections
- Tracks connection status per server: `Disconnected`, `Connecting`, `Connected`, `AuthRequired`, `Error`
- Exposes MCP tools and resources to the tool dispatch layer

### MCP Configuration
- Server configs live in `settings.json` under the `mcpServers` key
- Scope-aware merging: user-level and project-level configs are combined
- Signature computation for change detection: `runtime/src/mcp.rs` (`mcp_server_signature`)
- CCR proxy URL unwrapping: Extracts `mcp_url` query parameter from Anthropic proxy URLs

## LSP Integration

- Implementation: `runtime/src/lsp_client.rs`
- Registry-based architecture (`LspRegistry` with `Arc<Mutex<HashMap>>`)
- Supported actions: `Diagnostics`, `Hover`, `Definition`, `References`, `Completion`, `Symbols`, `Format`
- Connects to language servers via stdio subprocess (JSON-RPC over stdin/stdout)
- Used by tool dispatch for code intelligence features

## Web Tool Integrations

### Web Fetch
- Implementation: `tools/` crate (synchronous `reqwest::blocking::Client`)
- Used for fetching URLs on behalf of the LLM agent
- TLS via rustls

### PDF Extraction
- Implementation: `tools/src/pdf_extract.rs`
- Reads PDF files, locates `/Contents` stream objects
- Decompresses with `flate2` when stream uses `/FlateDecode`
- Extracts text between `BT`/`ET` markers

## Git Integration

- Implementation: `runtime/src/git_context.rs`
- Uses `git` CLI via `std::process::Command`
- Detects: branch name, recent 5 commits (hash + subject), staged files
- Additional git operations in `runtime/src/branch_lock.rs` (branch lock collision detection)
- `runtime/src/stale_base.rs` and `runtime/src/stale_branch.rs` for detecting stale git state
- Git commit provenance tracking in `runtime/src/lane_events.rs` (`LaneCommitProvenance`)

## File System Interactions

### File Operations (tools)
- Implementation: `runtime/src/file_ops.rs`
- `read_file` -- Reads file contents with optional offset/limit
- `write_file` -- Creates or overwrites files
- `edit_file` -- Applies structured patches (search/replace blocks)
- `glob_search` -- Finds files matching glob patterns (`glob 0.3`)
- `grep_search` -- Searches file contents with regex patterns (`regex 1`)

### Sandbox
- Implementation: `runtime/src/sandbox.rs`
- Filesystem isolation modes: `Off`, `WorkspaceOnly` (default), `AllowList`
- Network isolation: Optional via `network_isolation` config
- Namespace restrictions: Linux namespaces for process isolation
- Container detection: Checks `/.dockerenv`, `/run/.containerenv`, `/proc/1/cgroup`
- Mount control: `allowed_mounts` configuration
- Sandbox status detection inputs include env vars, docker/container markers, and cgroup info

## Process / Subprocess Management

### Bash Execution
- Implementation: `runtime/src/bash.rs`
- Input: `BashCommandInput` with command string, optional timeout, description, sandbox options
- Output: `BashCommandOutput` with stdout, stderr, exit code interpretation, background task support
- Uses `tokio::process::Command` for async subprocess management
- Timeout enforcement via `tokio::time::timeout`
- Background task support with `run_in_background` flag
- Sandbox integration: Commands can run in sandboxed environments
- Validation: `runtime/src/bash_validation.rs` for command safety checks

### MCP Subprocess
- Stdio transport spawns MCP server subprocesses via `tokio::process::Command`
- Manages child process lifecycle (stdin/stdout pipes, cleanup on drop)

### Mock API Server
- Implementation: `crates/mock-anthropic-service/`
- Standalone binary that simulates the Anthropic Messages API
- Binds to `127.0.0.1:0` (random port), prints `MOCK_ANTHROPIC_BASE_URL` for test consumption
- Used in integration tests via `dev-dependencies`

## Network Protocols

### HTTP/HTTPS
- Primary protocol for all API communication
- Client: `reqwest 0.12` with rustls-tls
- Used for: Anthropic API, OpenAI-compatible endpoints, web fetch tool, MCP SSE/HTTP transports

### Server-Sent Events (SSE)
- Streaming protocol for Anthropic and OpenAI-compatible APIs
- Implementation: `api/src/sse.rs` (`SseParser`) for Anthropic wire format
- OpenAI-compat has its own inline SSE parser (`OpenAiSseParser` in `openai_compat.rs`)
- Frame detection: `\n\n` or `\r\n\r\n` separators
- Handles chunked delivery, `data: [DONE]` termination, `event: ping` keepalive

### WebSocket
- Supported as MCP transport type (`McpServerConfig::Ws`)
- Implementation delegates to the MCP stdio/remote transport layer

### JSON-RPC over stdio
- Used for MCP client-server communication (LSP-style framing)
- Newline-delimited JSON messages over stdin/stdout pipes
- Implementation: `runtime/src/mcp_stdio.rs` (`JsonRpcRequest`, `JsonRpcResponse`)

### stdio
- Primary transport for MCP stdio servers
- Used for the built-in MCP server (`runtime/src/mcp_server.rs`)
- Framing: Newline-delimited JSON-RPC

## TUI / Terminal Libraries

### Line Editing
- `rustyline 15` -- REPL input with history, completion, Emacs/Vi keybindings

### Terminal Control
- `crossterm 0.28` -- Cross-platform terminal manipulation
  - Cursor control: save/restore position, move to column
  - Style: foreground colors, reset, bold/italic
  - Terminal: clear screen/line
  - Used in: `rusty-claude-cli/src/render.rs`

### Markdown Rendering
- `pulldown-cmark 0.13` -- Streaming CommonMark parser
  - Parses markdown events from assistant response stream
  - Handles code blocks, headings, links, tables, blockquotes, inline formatting

### Syntax Highlighting
- `syntect 5` -- Sublime-syntax-based highlighting
  - `SyntaxSet` for language detection
  - `ThemeSet` for color themes
  - `HighlightLines` for line-by-line highlighting
  - 24-bit terminal escape output via `as_24_bit_terminal_escaped`
  - Used for code blocks in the terminal renderer

## Plugin / Extension System

### Plugin Architecture
- Implementation: `plugins/src/lib.rs`
- Plugin types: `Builtin`, `Bundled`, `External`
- Manifest: `.claude-plugin/plugin.json` in each plugin directory
- Registry: `installed.json` tracks installed external plugins
- Settings: `settings.json` per plugin directory

### Plugin Lifecycle
- Manager: `PluginManager` in `plugins/src/lib.rs`
- Lifecycle: `runtime/src/plugin_lifecycle.rs`
- Discovery: Scans builtin, bundled, and external marketplaces
- Config: `PluginManagerConfig` controls install root, registry path, bundled root

### Plugin Hooks
- Implementation: `plugins/src/hooks.rs`, `runtime/src/hooks.rs`
- Hook events: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`
- Hook runner executes shell commands and parses output
- Results: `HookRunResult` with `denied`, `failed`, and `messages` fields
- Hook commands are configured in `settings.json` under `hooks` key

### Plugin Tools
- Plugins can contribute tool definitions via `PluginTool` type
- Tool specs are registered in the global tool registry

## Telemetry / Analytics

### Event System
- Implementation: `telemetry/src/lib.rs`
- Event types: `AnalyticsEvent`, `TelemetryEvent`
- Request profiling: `AnthropicRequestProfile` tracks API call metadata
- Session tracing: `SessionTracer` records HTTP requests, success/failure, latency
- Sinks: `JsonlTelemetrySink` (file-based), `MemoryTelemetrySink` (in-memory for testing)

### Client Identity
- `ClientIdentity` struct with `app_name`, `app_version`, `runtime` fields
- User-Agent header: `claude-code/{version}`
- Anthropic version header: `2023-06-01`
- Beta headers: `claude-code-20250219`, `prompt-caching-scope-2026-01-05`

### Cost Tracking
- Implementation: `runtime/src/usage.rs`
- Per-model pricing: `ModelPricing` struct (input/output/cache costs per million tokens)
- Cost estimation: `UsageCostEstimate` computed from `TokenUsage` + pricing
- Default pricing: Sonnet-tier rates ($15/$75/$18.75/$1.50 per million tokens)

## Configuration File Formats

### settings.json
- Schema name: `SettingsSchema`
- Locations (precedence order):
  1. `$CLAW_CONFIG_HOME/settings.json` (user scope)
  2. `$HOME/.claw/settings.json` (user scope, fallback)
  3. `.claw/settings.json` in CWD (project scope)
- Format: JSON with nested objects
- Validation: `runtime/src/config_validate.rs` detects unsupported format issues
- Scope-aware merging: project settings override user settings

### .env (credentials only)
- Location: CWD
- Format: Minimal `KEY=VALUE` grammar
- Supports comments (`#`), quoted values, `export` prefix
- Used only for credential discovery, not general configuration
- Parsed by `api/src/providers/mod.rs` (`parse_dotenv`)

### Config sections in settings.json
- `hooks` -- Pre/post tool use hook commands
- `plugins` -- Plugin settings (enabled list, maxOutputTokens, install paths)
- `mcpServers` -- MCP server configurations (stdio, SSE, HTTP, WS, SDK, managed proxy)
- `oauth` -- OAuth configuration (client_id, authorize_url, token_url, callback_port, scopes)
- `model` -- Default model selection
- `aliases` -- Model alias mappings
- `permissions` -- Permission mode and rules (allow/deny/ask patterns)
- `sandbox` -- Sandbox configuration (filesystem, network, namespace)
- `providerFallbacks` -- Fallback model chain for retryable failures
- `trustedRoots` -- Trusted certificate roots

### Session Files
- Extension: `.jsonl` (primary) or `.json` (legacy)
- Format: JSON Lines (one JSON object per line)
- Location: Project `.claw/` directory or specified session path

### Plugin Manifests
- Location: `.claude-plugin/plugin.json` within each plugin directory
- Contains: id, name, version, description, kind, source, default_enabled, hooks, tools

## CI/CD & Build

**Build system:**
- Cargo (no build system plugins)
- `build.rs` injects git SHA, target triple, and build date

**Linting:**
- `cargo fmt` (formatting)
- `cargo clippy --workspace --all-targets -- -D warnings` (lint enforcement)

**Testing:**
- `cargo test --workspace`
- Mock server: `mock-anthropic-service` binary for integration testing
- Environment-variable serialization guards in tests (EnvVarGuard pattern)

---

*Integration audit: 2026-04-14*
