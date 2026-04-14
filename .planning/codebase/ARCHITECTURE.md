# Architecture

**Analysis Date:** 2026-04-14

## Pattern Overview

**Overall:** Layered actor-loop architecture with a trait-based tool dispatch layer and a pluggable MCP (Model Context Protocol) server ecosystem.

**Key Characteristics:**
- Five-layer dependency hierarchy: `telemetry` / `plugins` (leaf) -> `runtime` (core) -> `api` / `commands` -> `tools` -> `rusty-claude-cli` (binary)
- The `ConversationRuntime` is the central coordinator: it owns the session, drives the model loop, dispatches tools, enforces permissions, and runs hooks
- Tool dispatch is trait-based (`ToolExecutor`) so the runtime is independent of any specific tool implementation
- MCP servers are both consumed (as tool sources) and exposed (as a stdio server `McpServer`) using the same JSON-RPC protocol
- Six global singleton registries (`TaskRegistry`, `TeamRegistry`, `CronRegistry`, `McpToolRegistry`, `LspRegistry`, `WorkerRegistry`) manage cross-invocation state via `OnceLock`

## High-Level System Diagram

```
                    +-------------------+
                    | rusty-claude-cli  |  (binary crate: REPL / one-shot entry)
                    | src/main.rs       |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
        +-----+-----+  +----+----+  +----+------+
        |  commands  |  |  tools  |  | compat-    |
        |  crate     |  |  crate  |  | harness    |
        +-----+------+  +----+----+  +------------+
              |              |
              +------+-------+
                     |
               +-----+------+
               |   runtime   |  (core primitives, traits, session, config)
               |   crate     |
               +--+----+-----+
                  |    |
          +-------+    +-------+
          |                   |
    +-----+------+     +-----+------+
    |    api     |     | telemetry  |
    |   crate    |     |   crate    |
    +-----+------+     +-----+------+
          |                   |
    +-----+------+     +-----+------+
    |  upstream  |     |   plugins  |
    | providers  |     |   crate    |
    +------------+     +------------+
```

## Crate Dependency Graph

```
telemetry  <---  runtime  <---  api
    ^              ^            ^
    |              |            |
    |         plugins           |
    |              ^            |
    |              |            |
    +--------------+            |
                   |            |
              commands           |
                   ^            |
                   |            |
                   +------+-----+
                          |
                        tools
                          ^
                          |
                   rusty-claude-cli ---> compat-harness
```

**Why this structure:**
- `telemetry` and `plugins` are leaf crates with zero internal workspace dependencies -- maximally reusable
- `runtime` depends only on `plugins` and `telemetry` -- it defines all core traits and data structures without knowing about HTTP clients or tool implementations
- `api` depends on `runtime` + `telemetry` -- it translates between wire protocols and runtime types
- `commands` depends on `runtime` + `plugins` -- slash command specs and handlers use runtime config/types
- `tools` depends on `api`, `commands`, `runtime`, and `plugins` -- it wires concrete tool implementations to the runtime's `ToolExecutor` trait
- `rusty-claude-cli` depends on everything -- it is the binary entry point that assembles all components
- `compat-harness` depends on `commands`, `tools`, and `runtime` -- it extracts TypeScript upstream manifests for parity checking
- `mock-anthropic-service` depends on `api` and `telemetry` -- a test-only HTTP server simulating the Anthropic API

## Layers

**Binary Layer (rusty-claude-cli):**
- Purpose: CLI argument parsing, REPL loop, terminal rendering, wiring all crates together
- Location: `crates/rusty-claude-cli/`
- Contains: `main.rs`, `init.rs`, `input.rs`, `render.rs`
- Depends on: All other crates
- Used by: End users via the `claw` binary

**Tool Layer (tools):**
- Purpose: Concrete tool implementations (Bash, Read, Write, Edit, Glob, Grep, etc.), tool dispatch, singleton registries
- Location: `crates/tools/`
- Contains: `lib.rs` (tool dispatch + `GlobalToolRegistry`), `lane_completion.rs`, `pdf_extract.rs`
- Depends on: `api`, `commands`, `runtime`, `plugins`
- Used by: `rusty-claude-cli`

**API Layer (api):**
- Purpose: HTTP client, SSE streaming parser, provider abstraction (Anthropic, OpenAI-compatible), request/response types
- Location: `crates/api/`
- Contains: `client.rs`, `sse.rs`, `types.rs`, `providers/`, `http_client.rs`, `prompt_cache.rs`, `error.rs`
- Depends on: `runtime`, `telemetry`
- Used by: `tools`, `mock-anthropic-service`

**Core Layer (runtime):**
- Purpose: Session persistence, config loading, permission evaluation, MCP plumbing, prompt assembly, conversation loop, tool-facing file ops
- Location: `crates/runtime/`
- Contains: `conversation.rs`, `session.rs`, `config.rs`, `permissions.rs`, `mcp_*.rs`, `hooks.rs`, `policy_engine.rs`, `sandbox.rs`, etc.
- Depends on: `plugins`, `telemetry`
- Used by: Nearly all other crates

**Support Layer (plugins, telemetry):**
- Purpose: Plugin manifest/model types and discovery; telemetry event types and sinks
- Location: `crates/plugins/`, `crates/telemetry/`
- Depends on: `serde`, `serde_json` only (no workspace deps)
- Used by: `runtime`, `api`, `commands`, `tools`

## Core Runtime Loop (ConversationRuntime)

The `ConversationRuntime<C, T>` in `crates/runtime/src/conversation.rs` is the heart of the system. It is generic over:
- `C: ApiClient` -- the streaming model client
- `T: ToolExecutor` -- the tool dispatcher

### Turn Flow

```
User Input
    |
    v
Session.push_user_text()  -->  JSONL append
    |
    v
+--- loop (max_iterations) -------------------------------------------+
|                                                                      |
|  1. Build ApiRequest { system_prompt, messages }                     |
|  2. api_client.stream(request) -> Vec<AssistantEvent>               |
|  3. build_assistant_message(events) -> ConversationMessage          |
|     (accumulates TextDelta, ToolUse blocks; requires MessageStop)    |
|  4. session.push_message(assistant_message)                          |
|                                                                      |
|  5. For each pending ToolUse block:                                  |
|     a. run_pre_tool_use_hook(tool_name, input)                       |
|        --> may override/deny/cancel                                  |
|     b. permission_policy.authorize_with_context(...)                  |
|        --> Allow | Deny                                              |
|     c. If Allowed:                                                   |
|        - tool_executor.execute(tool_name, input) -> output           |
|        - run_post_tool_use_hook / run_post_tool_use_failure_hook     |
|        - session.push_message(tool_result)                           |
|     d. If Denied:                                                    |
|        - session.push_message(error_tool_result)                     |
|                                                                      |
|  6. If no ToolUse blocks in assistant message --> break              |
+----------------------------------------------------------------------+
    |
    v
maybe_auto_compact()  -->  if cumulative input tokens > threshold
    |
    v
Return TurnSummary { assistant_messages, tool_results, usage, iterations }
```

### Key Design Decisions
- The loop is **synchronous** from the runtime's perspective -- `ApiClient::stream()` returns a fully-collected `Vec<AssistantEvent>`, not an async stream
- Tool execution happens **inline** within the loop (not async/background)
- Hooks wrap every tool invocation: `PreToolUse` before, `PostToolUse` after success, `PostToolUseFailure` after errors
- Permission evaluation uses a three-tier system: deny rules -> mode check -> ask rules -> prompt

## Data Flow: User Input -> API Call -> Tool Dispatch -> Response

```
1. CLI parses input (REPL or --prompt flag)
2. LiveCli builds:
   - System prompt (from CLAUDE.md, git context, OS info)
   - RuntimeConfig (from .claw/settings.json cascade)
   - PermissionPolicy (from config + CLI flags)
   - ToolExecutor (GlobalToolRegistry dispatch)
   - ApiClient adapter (bridges ProviderClient -> runtime ApiClient trait)
3. ConversationRuntime.run_turn(user_input, prompter) executes:
   a. Assemble ApiRequest with current session messages
   b. HTTP POST to upstream provider (Anthropic/xAI/OpenAI)
   c. Parse SSE stream -> StreamEvent -> AssistantEvent
   d. Extract tool_use blocks, run through permission + hook pipeline
   e. Execute tools, feed results back into session
   f. Repeat until model returns text-only response (no tool_use)
4. CLI renders AssistantEvents to terminal (MarkdownStreamState)
5. Session persisted to JSONL
```

## Key Abstractions

### ConversationRuntime (`crates/runtime/src/conversation.rs`)
- Coordinates the model loop, tool execution, hooks, and session updates
- Generic over `ApiClient` and `ToolExecutor` traits
- Tracks cumulative `UsageTracker` and optional `SessionTracer`
- Supports auto-compaction when token threshold is crossed
- Provides session forking via `fork_session()`

### Session (`crates/runtime/src/session.rs`)
- Persisted conversational state: messages, compaction metadata, fork provenance, workspace root
- Dual serialization: JSON (legacy) and JSONL (current, append-friendly)
- JSONL record types: `session_meta`, `message`, `compaction`, `prompt_history`
- Atomic writes via temp-file + rename
- Log rotation at 256KB with max 3 rotated files
- Monotonic timestamps via `AtomicU64` CAS loop

### RuntimeConfig (`crates/runtime/src/config.rs`)
- Merged from a precedence chain: User `.claw.json` < User `settings.json` < Project `.claw.json` < Project `.claw/settings.json` < Local `.claw/settings.local.json`
- Deep-merges JSON objects (later scopes win for leaf keys)
- Validates against known schema keys with line-number diagnostics
- Extracts typed feature configs: hooks, plugins, MCP servers, OAuth, sandbox, permission rules, model aliases, provider fallbacks

### ToolExecutor Trait (`crates/runtime/src/conversation.rs`)
```rust
pub trait ToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError>;
}
```
- Implemented by `GlobalToolRegistry` (in `tools` crate) for production
- `StaticToolExecutor` provided for tests (register closures by name)
- The runtime never knows about specific tools -- all dispatch is name-based

### ApiClient Trait (`crates/runtime/src/conversation.rs`)
```rust
pub trait ApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>;
}
```
- Bridges the async `api::ProviderClient` to the synchronous runtime loop
- The CLI's adapter blocks on tokio `block_in_place()` to collect streamed events

### PermissionPolicy and PermissionEnforcer (`crates/runtime/src/permissions.rs`, `permission_enforcer.rs`)
- `PermissionPolicy`: evaluates tool authorization based on active mode, tool requirements, allow/deny/ask rules, and hook overrides
- Permission modes (ordered by privilege): `ReadOnly` < `WorkspaceWrite` < `DangerFullAccess`
- Rule syntax: `ToolName(exact_match)`, `ToolName(prefix:*)`, or bare `ToolName`
- `PermissionEnforcer`: wraps `PermissionPolicy` with a simpler `check()` API for non-prompting contexts
- `PermissionPrompter` trait: interactive approval interface implemented by the CLI

### McpServer / McpServerManager (`crates/runtime/src/mcp_server.rs`, `mcp_stdio.rs`)
- `McpServer`: minimal MCP stdio server (JSON-RPC over Content-Length framed transport)
  - Handles `initialize`, `tools/list`, `tools/call`
  - Protocol version `2025-03-26`
  - Delegates tool execution to a caller-supplied `ToolCallHandler` closure
- `McpServerManager`: spawns and manages MCP server child processes
  - JSON-RPC client for stdio transport
  - Tool discovery, resource listing, OAuth flows
  - Transport types: Stdio, SSE, HTTP, WebSocket, SDK, ManagedProxy
- `McpToolRegistry`: singleton bridge connecting MCP server state to tool dispatch
  - Tracks connection status, discovered tools/resources per server

### PluginManager / PluginLifecycle (`crates/plugins/src/lib.rs`, `crates/runtime/src/plugin_lifecycle.rs`)
- `PluginManager` (plugins crate): discovers, validates, and loads plugin manifests from builtin/bundled/external sources
  - Manifest format: `plugin.json` with hooks, lifecycle commands, tool definitions, permissions
  - Plugin kinds: `Builtin`, `Bundled`, `External`
- `PluginLifecycle` (runtime crate): runtime state machine for plugin servers
  - States: `Unconfigured` -> `Validated` -> `Starting` -> `Healthy` / `Degraded` / `Failed` -> `Stopped`
  - Bridges plugin health to the MCP lifecycle validator

## Singleton Registries

All six registries in `crates/tools/src/lib.rs` use `OnceLock` for process-global initialization:

| Registry | Source | Purpose |
|----------|--------|---------|
| `TaskRegistry` | `crates/runtime/src/task_registry.rs` | Sub-agent task lifecycle (Created/Running/Completed/Failed/Stopped) |
| `TeamRegistry` | `crates/runtime/src/team_cron_registry.rs` | Team definitions for agent collaboration |
| `CronRegistry` | `crates/runtime/src/team_cron_registry.rs` | Scheduled recurring task management |
| `McpToolRegistry` | `crates/runtime/src/mcp_tool_bridge.rs` | MCP server connection state, tool/resource discovery |
| `LspRegistry` | `crates/runtime/src/lsp_client.rs` | Language Server Protocol client connections |
| `WorkerRegistry` | `crates/runtime/src/worker_boot.rs` | Worker process state machine (Spawning/TrustRequired/ReadyForPrompt/Running/Finished/Failed) |

## Streaming Architecture

**SSE Parsing** (`crates/api/src/sse.rs`):
- `SseParser`: stateful byte-buffer parser that handles partial frames across TCP chunks
- Splits on `\n\n` or `\r\n\r\n` boundaries
- Parses `event:` and `data:` fields, joins multi-line data
- Ignores `ping` events and `[DONE]` sentinel
- Deserializes data payloads into `StreamEvent` enum

**Provider Streaming** (`crates/api/src/providers/`):
- `AnthropicClient`: HTTP POST with SSE response body, yields `StreamEvent` variants
- `OpenAiCompatClient`: OpenAI-format SSE for xAI, OpenAI, and DashScope
- `ProviderClient` enum wraps both, providing `stream_message()` -> `MessageStream`
- `MessageStream` is an async iterator (`next_event()`)

**Bridging to Runtime** (`rusty-claude-cli/src/main.rs`):
- The CLI adapter implements `runtime::ApiClient` by calling `ProviderClient::stream_message()` inside `tokio::task::block_in_place()`
- Collects all `StreamEvent` into `Vec<AssistantEvent>` before returning to the synchronous runtime loop
- During collection, text deltas are rendered incrementally to the terminal via `TerminalRenderer`

**Incremental Rendering** (`crates/rusty-claude-cli/src/render.rs`):
- `TerminalRenderer`: wraps terminal write with ANSI escape codes for cursor control
- `MarkdownStreamState`: accumulates text deltas, flushes complete paragraphs
- `Spinner`: renders a progress indicator during tool execution

## REPL vs One-Shot Prompt Modes

**REPL Mode** (`CliAction::Repl`):
- Entered when no positional prompt argument is provided
- Uses `rustyline` for line editing with history
- Renders a `claw>` prompt
- Supports slash commands (prefixed with `/`)
- Session persists across turns (JSONL append)
- Ctrl+C / Ctrl+D exits cleanly

**One-Shot Mode** (`CliAction::Prompt`):
- Entered when a positional prompt argument is provided
- Runs a single `ConversationRuntime::run_turn()` then exits
- Supports `--print` flag for non-interactive output
- Supports `--output-format json` for machine-readable output
- Supports `--resume <session-path>` to continue a previous session
- Stdin piping: when stdin is not a terminal and mode is `DangerFullAccess`, piped content is merged into the prompt

## Slash Commands

Defined in `crates/commands/src/lib.rs` as a static `SLASH_COMMAND_SPECS` array. Each `SlashCommandSpec` has:
- `name`: the command identifier (e.g. `help`, `status`, `compact`, `model`, `mcp`)
- `aliases`: alternative names
- `summary`: help text
- `argument_hint`: optional parameter description
- `resume_supported`: whether the command works in session-resume mode

Slash commands are intercepted by the CLI before reaching the runtime. They either:
1. Return local output (e.g. `/status`, `/cost`, `/version`)
2. Mutate runtime state (e.g. `/model`, `/permissions`, `/clear`)
3. Delegate to the runtime as a normal user turn (e.g. `/commit`, `/pr`, `/bughunter`)

The `resume_supported` flag controls which commands can be replayed when resuming a session via `--resume`.

## Error Handling

**Strategy:** Explicit error types at each layer, propagated via `Result<T, E>`.

**Patterns:**
- `RuntimeError` / `ToolError` in the runtime layer -- string-based error messages
- `ApiError` in the API layer -- structured with provider/model context for deserialization failures
- `SessionError` in the session layer -- `Io`, `Json`, `Format` variants
- `ConfigError` in the config layer -- `Io`, `Parse` variants
- The CLI's `run()` returns `Result<(), Box<dyn std::error::Error>>` for top-level error handling
- Hook failures are non-fatal: they produce error feedback merged into tool output rather than aborting the turn

## Cross-Cutting Concerns

**Logging:** No logging framework. Errors and status are written to stderr via `eprintln!`. The `SessionTracer` in telemetry provides structured trace events for session lifecycle.

**Validation:**
- Config validation: `config_validate.rs` with `ValidationResult` containing errors/warnings with line numbers and field paths
- Bash validation: `bash_validation.rs` classifies commands by danger level
- Branch locking: `branch_lock.rs` detects collisions between parallel agents
- Stale branch detection: `stale_branch.rs` checks freshness against upstream

**Authentication:**
- Anthropic: API key from `ANTHROPIC_API_KEY` env, or OAuth token from `~/.claude/oauth.json`
- OpenAI: API key from `OPENAI_API_KEY` env
- xAI: API key from `XAI_API_KEY` env
- DashScope: API key from `DASHSCOPE_API_KEY` env
- OAuth PKCE flow: `crates/runtime/src/oauth.rs` -- authorization code with S256 challenge
- `resolve_startup_auth_source()` selects auth method based on available credentials

**Sandbox:** `crates/runtime/src/sandbox.rs` provides Linux namespace-based isolation detection and command wrapping. Controlled by `SandboxConfig` from settings.

---

*Architecture analysis: 2026-04-14*
