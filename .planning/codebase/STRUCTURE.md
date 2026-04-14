# Codebase Structure

**Analysis Date:** 2026-04-14

## Directory Layout

```
rust/
├── Cargo.toml              # Workspace root (9 members, resolver 2)
├── Cargo.lock              # Dependency lockfile
├── crates/                 # All workspace crates
│   ├── api/                # HTTP client, SSE parsing, provider abstraction
│   ├── commands/           # Slash command specs and handlers
│   ├── compat-harness/     # Upstream TypeScript manifest extraction
│   ├── mock-anthropic-service/  # Test-only mock API server
│   ├── plugins/            # Plugin manifest types and discovery
│   ├── runtime/            # Core runtime primitives (session, config, permissions, MCP)
│   ├── rusty-claude-cli/   # Binary crate: CLI entry point
│   ├── telemetry/          # Telemetry event types and sinks
│   └── tools/              # Concrete tool implementations and dispatch
├── scripts/                # Build/utility scripts
├── MOCK_PARITY_HARNESS.md  # Parity testing documentation
├── PARITY.md               # Upstream compatibility notes
├── README.md               # Project readme
├── TUI-ENHANCEMENT-PLAN.md # Terminal UI improvement plan
└── USAGE.md                # User-facing usage guide
```

## Crate Map

### `runtime` (~45 source files, ~18,000 LOC)
- **Responsibility:** Core primitives for session persistence, config loading, permission evaluation, prompt assembly, MCP plumbing, tool-facing file operations, the conversation loop
- **Key public API:**
  - `ConversationRuntime<C, T>` -- model loop coordinator
  - `Session` -- persisted conversational state
  - `ConfigLoader` / `RuntimeConfig` -- layered config merging
  - `PermissionPolicy` / `PermissionMode` -- authorization engine
  - `McpServer` / `McpServerManager` -- MCP server/client
  - `ToolExecutor` / `ApiClient` -- core traits
  - `BashCommandInput`/`BashCommandOutput`, file ops (`read_file`, `write_file`, `edit_file`, `glob_search`, `grep_search`)
- **Internal dependencies:** `plugins`, `telemetry`
- **External dependencies:** `sha2`, `glob`, `regex`, `serde`, `serde_json`, `tokio`, `walkdir`

### `tools` (~3 source files, ~12,000 LOC)
- **Responsibility:** Concrete tool implementations, global tool registry, tool dispatch, singleton registries
- **Key public API:**
  - `execute_tool()` -- main tool dispatch function
  - `mvp_tool_specs()` -- returns built-in tool specifications
  - `GlobalToolRegistry` -- unified registry for built-in + plugin + runtime tools
  - `ToolManifestEntry`, `ToolSource`, `ToolSpec` -- tool metadata types
- **Internal dependencies:** `api`, `commands`, `runtime`, `plugins`
- **External dependencies:** `api`, `commands`, `flate2`, `plugins`, `runtime`, `reqwest`, `serde`, `serde_json`, `tokio`

### `commands` (~1 source file, ~5,600 LOC)
- **Responsibility:** Slash command specifications, command handlers, skill system
- **Key public API:**
  - `slash_command_specs()` -- static list of `SlashCommandSpec`
  - `handle_mcp_slash_command()` / `handle_mcp_slash_command_json()`
  - `handle_plugins_slash_command()`
  - `handle_skills_slash_command()`
  - `CommandRegistry`, `CommandManifestEntry`
- **Internal dependencies:** `runtime`, `plugins`
- **External dependencies:** `plugins`, `runtime`, `serde_json`

### `plugins` (~3 source files, ~4,200 LOC)
- **Responsibility:** Plugin manifest types, discovery, loading, and hook execution
- **Key public API:**
  - `PluginManager` -- discovers and loads plugins
  - `PluginManifest`, `PluginMetadata`, `PluginTool` -- manifest types
  - `PluginKind` (Builtin/Bundled/External)
  - `PluginHooks`, `PluginLifecycle` -- hook/lifecycle definitions
  - `PluginError`, `PluginLoadFailure`, `PluginSummary`
- **Internal dependencies:** None (leaf crate)
- **External dependencies:** `serde`, `serde_json`

### `api` (~7 source files, ~7,500 LOC)
- **Responsibility:** HTTP client, SSE streaming parser, provider abstraction (Anthropic, OpenAI-compatible), request/response types, prompt caching
- **Key public API:**
  - `ProviderClient` (enum: Anthropic/Xai/OpenAi) -- unified provider interface
  - `MessageStream` -- async event iterator
  - `SseParser` / `parse_frame()` -- SSE frame parsing
  - `MessageRequest`, `MessageResponse`, `StreamEvent` -- wire types
  - `PromptCache` -- token caching for Anthropic
  - `build_http_client()` -- configured reqwest client
- **Internal dependencies:** `runtime`, `telemetry`
- **External dependencies:** `reqwest`, `runtime`, `serde`, `serde_json`, `telemetry`, `tokio`

### `rusty-claude-cli` (~4 source files, ~13,500 LOC)
- **Responsibility:** Binary entry point, CLI argument parsing, REPL loop, terminal rendering, assembly of all runtime components
- **Key files:**
  - `src/main.rs` -- argument parsing, REPL loop, `LiveCli` struct, `run_turn_with_output()`
  - `src/init.rs` -- `CLAUDE.md` initialization
  - `src/input.rs` -- stdin reading utilities
  - `src/render.rs` -- `TerminalRenderer`, `MarkdownStreamState`, `Spinner`
  - `build.rs` -- build-time constants (date, git sha)
- **Binary name:** `claw`
- **Internal dependencies:** All other crates
- **External dependencies:** `crossterm`, `pulldown-cmark`, `rustyline`, `syntect`, `tokio`

### `telemetry` (~1 source file, ~1,300 LOC)
- **Responsibility:** Telemetry event types, in-memory and JSONL sinks, session tracing
- **Key public API:**
  - `TelemetryEvent` (enum: Analytics/SessionTrace)
  - `TelemetrySink` trait, `MemoryTelemetrySink`, `JsonlTelemetrySink`
  - `SessionTracer` -- attaches trace events to a session
  - `ClientIdentity`, `AnthropicRequestProfile`
  - `DEFAULT_ANTHROPIC_VERSION`, `DEFAULT_AGENTIC_BETA`
- **Internal dependencies:** None (leaf crate)
- **External dependencies:** `serde`, `serde_json`

### `compat-harness` (~1 source file, ~900 LOC)
- **Responsibility:** Extracts command and tool manifests from upstream TypeScript source for parity checking
- **Key public API:**
  - `extract_manifest()` -- parses TypeScript imports
  - `UpstreamPaths` -- resolves paths to upstream source files
  - `ExtractedManifest` -- extracted commands, tools, bootstrap plan
- **Internal dependencies:** `commands`, `tools`, `runtime`
- **External dependencies:** None beyond workspace crates

### `mock-anthropic-service` (~2 source files, ~1,500 LOC)
- **Responsibility:** Test-only HTTP server that simulates the Anthropic Messages API
- **Binary name:** `mock-anthropic-service`
- **Key public API:**
  - `lib.rs` -- server setup functions
  - `main.rs` -- standalone server binary
- **Internal dependencies:** `api`, `telemetry`
- **External dependencies:** `api`, `serde_json`, `tokio`

## Module Structure Within Key Crates

### `runtime` crate (`crates/runtime/src/`)
```
lib.rs                    # Public re-exports (the crate's API surface)
conversation.rs           # ConversationRuntime<C,T>, ApiClient trait, ToolExecutor trait
session.rs                # Session, ConversationMessage, ContentBlock, JSONL persistence
session_control.rs        # SessionStore, SessionHandle (per-worktree session namespacing)
config.rs                 # ConfigLoader, RuntimeConfig, McpServerConfig variants
config_validate.rs        # ConfigDiagnostic, ValidationResult, schema validation
permissions.rs            # PermissionPolicy, PermissionMode, PermissionRule
permission_enforcer.rs    # PermissionEnforcer, EnforcementResult
prompt.rs                 # SystemPromptBuilder, ProjectContext, load_system_prompt
hooks.rs                  # HookRunner, HookEvent, HookAbortSignal
mcp_server.rs             # McpServer (stdio JSON-RPC server)
mcp_stdio.rs              # McpServerManager, McpStdioProcess, JSON-RPC types
mcp_client.rs             # McpClientTransport, McpClientBootstrap
mcp_tool_bridge.rs        # McpToolRegistry, McpServerState
mcp_lifecycle_hardened.rs # McpLifecycleState, McpLifecycleValidator
mcp.rs                    # MCP naming utilities (prefixes, normalization)
plugin_lifecycle.rs       # PluginState, PluginLifecycle, ServerHealth
bash.rs                   # execute_bash()
bash_validation.rs        # Bash command danger classification
file_ops.rs               # read_file, write_file, edit_file, glob_search, grep_search
git_context.rs            # GitContext, GitCommitEntry
sandbox.rs                # SandboxConfig, SandboxStatus, container detection
sse.rs                    # IncrementalSseParser (runtime-level SSE parsing)
compact.rs                # Session compaction logic
summary_compression.rs    # Text compression for summaries
usage.rs                  # TokenUsage, UsageTracker, ModelPricing, format_usd
task_registry.rs          # TaskRegistry (singleton)
team_cron_registry.rs     # TeamRegistry, CronRegistry (singletons)
worker_boot.rs            # WorkerRegistry, WorkerStatus (singleton)
lane_events.rs            # LaneEvent, LaneEventName, LaneFailureClass
policy_engine.rs          # PolicyEngine, PolicyRule, PolicyCondition, LaneContext
recovery_recipes.rs       # Failure recovery patterns
stale_base.rs             # Base commit staleness detection
stale_branch.rs           # Branch freshness checking
branch_lock.rs            # Branch lock collision detection
green_contract.rs         # Green-level evaluation
task_packet.rs            # TaskPacket validation
remote.rs                 # Remote session context, proxy bootstrap
oauth.rs                  # OAuth PKCE flow helpers
json.rs                   # Minimal JSON parser (JsonValue)
bootstrap.rs              # BootstrapPhase, BootstrapPlan
```

### `api` crate (`crates/api/src/`)
```
lib.rs                    # Public re-exports
client.rs                 # ProviderClient enum (Anthropic/Xai/OpenAi), MessageStream
types.rs                  # MessageRequest, MessageResponse, StreamEvent, InputMessage, etc.
sse.rs                    # SseParser, parse_frame (byte-level SSE parsing)
error.rs                  # ApiError
http_client.rs            # build_http_client, ProxyConfig
prompt_cache.rs           # PromptCache, PromptCacheConfig, PromptCacheRecord
providers/
  mod.rs                  # detect_provider_kind, resolve_model_alias, max_tokens_for_model
  anthropic.rs            # AnthropicClient, AuthSource, streaming
  openai_compat.rs        # OpenAiCompatClient, OpenAiCompatConfig
```

### `tools` crate (`crates/tools/src/`)
```
lib.rs                    # Tool dispatch, GlobalToolRegistry, singleton accessors
lane_completion.rs        # Lane completion detection
pdf_extract.rs            # PDF text extraction (flate2 decompression)
```

## Key Types and Their Roles (Cross-Crate)

| Type | Crate | Cross-Crate Usage |
|------|-------|-------------------|
| `ConversationRuntime<C, T>` | runtime | Constructed in `rusty-claude-cli`, holds Session + ApiClient + ToolExecutor |
| `Session` | runtime | Passed between CLI and runtime, persisted to disk |
| `RuntimeConfig` | runtime | Loaded in CLI, consumed by runtime subsystems |
| `PermissionPolicy` | runtime | Created in CLI from config, used by ConversationRuntime |
| `ToolExecutor` trait | runtime | Implemented by tools::GlobalToolRegistry |
| `ApiClient` trait | runtime | Implemented by CLI adapter over api::ProviderClient |
| `ProviderClient` | api | Used by CLI to talk to upstream providers |
| `StreamEvent` | api | Parsed from SSE, translated to runtime::AssistantEvent |
| `AssistantEvent` | runtime | Emitted by ApiClient, consumed by ConversationRuntime |
| `McpServerManager` | runtime | Managed in CLI for MCP tool discovery |
| `McpToolRegistry` | runtime | Singleton in tools crate, bridges MCP to tool dispatch |
| `PluginManager` | plugins | Used in CLI to discover and load plugins |
| `PluginTool` | plugins | Registered in GlobalToolRegistry |
| `SessionTracer` | telemetry | Attached to ConversationRuntime for trace events |
| `TaskPacket` | runtime | Validated and stored in TaskRegistry |

## Entry Points

**Binary entry:** `crates/rusty-claude-cli/src/main.rs`
- Defines `fn main()` -> `fn run()` -> argument dispatch
- `CliAction` enum routes to REPL, one-shot prompt, or diagnostic subcommands
- `LiveCli` struct is the main orchestrator for interactive and one-shot modes

**Library entry points (lib.rs re-exports):**
- `crates/runtime/src/lib.rs` -- 170+ public items, the largest API surface
- `crates/api/src/lib.rs` -- provider client, SSE parser, wire types
- `crates/tools/src/lib.rs` -- `execute_tool()`, `mvp_tool_specs()`, `GlobalToolRegistry`
- `crates/commands/src/lib.rs` -- slash command specs and handlers
- `crates/plugins/src/lib.rs` -- `PluginManager`, manifest types
- `crates/telemetry/src/lib.rs` -- event types, sinks, tracing

## Test Organization Strategy

### Per-Crate Unit Tests
Most crates use inline `#[cfg(test)] mod tests` blocks at the bottom of source files. This is the dominant pattern.

**runtime** (~3,500 LOC of tests across files):
- `conversation.rs` -- 15+ tests: end-to-end turn loop, hook integration, permission denial, auto-compaction, session persistence, health probes
- `session.rs` -- 15+ tests: JSONL round-trip, legacy JSON loading, fork metadata, log rotation, timestamp monotonicity
- `config.rs` -- 15+ tests: config merging, MCP server parsing, permission mode aliases, hook config merge, plugin config, sandbox config, provider fallbacks, validation diagnostics
- `permissions.rs` -- 10+ tests: mode escalation, rule matching, hook overrides, prompt delegation
- `sse.rs` (api) -- 8 tests: frame parsing, chunked streams, ping/DONE handling, thinking blocks
- `mcp_server.rs` -- 5 tests: initialize, tools/list, tools/call, error surfacing, unknown method

### Integration Test Files
- `crates/api/tests/` -- `client_integration.rs`, `openai_compat_integration.rs`, `provider_client_integration.rs`, `proxy_integration.rs`
- `crates/runtime/tests/integration_tests.rs`
- `crates/rusty-claude-cli/tests/` -- `cli_flags_and_config_defaults.rs`, `compact_output.rs`, `mock_parity_harness.rs`, `output_format_contract.rs`, `resume_slash_commands.rs`

### Test Isolation
- `crates/runtime/src/lib.rs` provides `test_env_lock()` -- a process-global mutex to prevent test interference from shared environment state
- `crates/plugins/src/test_isolation.rs` -- test isolation utilities for plugin tests
- Temp directories are created with nanosecond-precision timestamps for uniqueness

### Mock Infrastructure
- `mock-anthropic-service` crate: standalone HTTP server simulating Anthropic API responses
- `ScriptedApiClient` (in conversation tests): in-memory ApiClient that returns predetermined event sequences
- `StaticToolExecutor` (in conversation tests): in-memory ToolExecutor with registered closures

## How the Binary Crate Wires Everything Together

`crates/rusty-claude-cli/src/main.rs` (~11,800 LOC) is the assembly point:

1. **Config loading:** `ConfigLoader::default_for(cwd).load()` -> `RuntimeConfig`
2. **Plugin loading:** `PluginManager::new(config).load()` -> `Vec<PluginTool>`
3. **MCP initialization:** `McpServerManager::from_config()` -> spawn stdio processes, discover tools
4. **Global registries:** `global_mcp_registry()`, `global_lsp_registry()`, `global_task_registry()`, etc. (OnceLock singletons)
5. **Tool registry assembly:** `GlobalToolRegistry::builtin().with_plugin_tools(plugins).with_runtime_tools(mcp_tools).with_enforcer(enforcer)`
6. **Provider client:** `ProviderClient::from_model_with_anthropic_auth(model, auth)` -> `ProviderClient`
7. **ApiClient adapter:** A closure or struct that bridges `ProviderClient::stream_message()` to `runtime::ApiClient::stream()`
8. **System prompt:** `SystemPromptBuilder::new().with_project_context(ctx).with_os(...).build()` -> `Vec<String>`
9. **Permission policy:** `PermissionPolicy::new(mode).with_tool_requirements(...).with_permission_rules(...)`
10. **ConversationRuntime:** `ConversationRuntime::new(session, api_client, tool_executor, policy, system_prompt)`
11. **REPL/one-shot loop:** Calls `runtime.run_turn(input, prompter)`, renders events via `TerminalRenderer`

## Naming Conventions

**Files:**
- `snake_case.rs` for all source files
- `mod.rs` not used; all modules are file-based (`module_name.rs`)
- Test files: `*_test.rs`, `*_tests.rs`, or inline `mod tests`

**Crates:**
- `kebab-case` directory names: `rusty-claude-cli`, `compat-harness`, `mock-anthropic-service`
- Crate names in `Cargo.toml` use `kebab-case` for the package but `snake_case` for the library target

**Types:**
- `PascalCase` for all types (structs, enums, traits)
- `UPPER_SNAKE_CASE` for constants

**Functions/Methods:**
- `snake_case` for all functions and methods

## Where to Add New Code

**New built-in tool:**
- Add tool spec to `mvp_tool_specs()` in `crates/tools/src/lib.rs`
- Add dispatch arm to `execute_tool()` in `crates/tools/src/lib.rs`
- Add `ToolDefinition` to the tool list sent to the provider
- Tests: inline `#[cfg(test)]` in `crates/tools/src/lib.rs`

**New slash command:**
- Add `SlashCommandSpec` entry to `SLASH_COMMAND_SPECS` in `crates/commands/src/lib.rs`
- Add handler function in `crates/commands/src/lib.rs`
- Wire dispatch in `crates/rusty-claude-cli/src/main.rs` (REPL slash command matching)
- Tests: inline in `crates/commands/src/lib.rs`

**New runtime primitive:**
- Add source file to `crates/runtime/src/`
- Add `mod` declaration and `pub use` in `crates/runtime/src/lib.rs`
- Tests: inline `#[cfg(test)]` in the new file

**New provider:**
- Add variant to `ProviderKind` in `crates/api/src/providers/mod.rs`
- Add client implementation in `crates/api/src/providers/`
- Add variant to `ProviderClient` in `crates/api/src/client.rs`
- Tests: inline in the provider file, integration test in `crates/api/tests/`

**New singleton registry:**
- Define struct in `crates/runtime/src/`
- Add `OnceLock` accessor in `crates/tools/src/lib.rs`
- Tests: inline in the definition file

## Special Directories

**`crates/mock-anthropic-service/`:**
- Purpose: Standalone HTTP server for integration testing
- Generated: No
- Committed: Yes
- Has its own binary target (`mock-anthropic-service`)

**`scripts/`:**
- Purpose: Build and utility scripts
- Generated: No
- Committed: Yes

---

*Structure analysis: 2026-04-14*
