# Coding Conventions

**Analysis Date:** 2026-04-14

## Naming Patterns

**Files:**
- `snake_case.rs` for all source files
- `lib.rs` for crate roots; no `mod.rs` used for submodules
- Integration test files: `*_integration.rs` in `tests/` directories (e.g., `client_integration.rs`, `provider_client_integration.rs`)
- Domain-specific test files: `mock_parity_harness.rs`, `cli_flags_and_config_defaults.rs`, `output_format_contract.rs`

**Functions:**
- `snake_case` throughout
- Predicate functions use `is_`, `has_`, `should_` prefixes (e.g., `is_retryable()`, `is_context_window_failure()`, `should_compact()`)
- Constructor methods use `new()` for primary, `with_*()` for builder-pattern (e.g., `AnthropicRequestProfile::new()`, `.with_beta()`, `.with_extra_body()`)
- Free functions used for operations (e.g., `load_system_prompt()`, `compact_session()`, `execute_bash()`)
- Builder-pattern chaining: `pub fn with_x(mut self, ...) -> Self { ... self }`

**Variables:**
- `snake_case` throughout
- Constants: `SCREAMING_SNAKE_CASE` (e.g., `DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD`, `SESSION_VERSION`)
- Module-level `const` values for thresholds, env var names, and string literals

**Types:**
- Structs: `PascalCase` (e.g., `RuntimeConfig`, `CompactionResult`, `CapturedRequest`)
- Enums: `PascalCase` variants (e.g., `PermissionMode::ReadOnly`, `HookEvent::PreToolUse`)
- Enums with serde use `#[serde(rename_all = "snake_case")]` for wire format
- Type aliases: `PascalCase` (e.g., `GreenLevel = u8`, `HookPermissionDecision = PermissionOverride`)
- Newtype-style structs with private fields for encapsulation

## Code Style

**Formatting:**
- Tool: `cargo fmt` (Rust standard formatter)
- No custom `rustfmt.toml` or `.rustfmt.toml` files -- default rustfmt settings apply
- Edition 2021 across all crates

**Linting:**
- Tool: `cargo clippy --workspace --all-targets -- -D warnings`
- Workspace-level lints in `Cargo.toml`:
  - `clippy::all = "warn"`, `clippy::pedantic = "warn"`
  - Allowed: `clippy::module_name_repetitions`, `clippy::missing_panics_doc`, `clippy::missing_errors_doc`
- `unsafe_code = "forbid"` at workspace level -- zero unsafe code permitted
- `publish = false` on all crates
- Per-file `#[allow(...)]` annotations used sparingly for `clippy::too_many_lines`, `clippy::struct_excessive_bools`, `clippy::await_holding_lock`

## Import Organization

**Order:**
1. `std` crate imports
2. External crate imports (`serde`, `tokio`, `reqwest`)
3. Workspace-internal crate imports (`runtime::`, `api::`, `plugins::`)
4. `crate::` relative imports (within same crate)

**Path Aliases:**
- No path aliases configured; all imports use crate names directly

**Re-exports:**
- `pub use` in `lib.rs` re-exports from submodules extensively
- Pattern: `mod foo;` (private module) + `pub use foo::{Bar, Baz};` (public re-exports)
- Selective re-export: only the public API surface is re-exported, internals stay private
- Example from `crates/runtime/src/lib.rs`: `mod bash;` (private) + `pub use bash::{execute_bash, BashCommandInput, BashCommandOutput};`

## Error Handling

**Pattern:** Custom error enums per domain, NOT `anyhow`.

**Error Types:**
- `api::ApiError` -- rich enum with structured variants (see `crates/api/src/error.rs`)
  - `MissingCredentials`, `ContextWindowExceeded`, `Auth`, `Http`, `Io`, `Json`, `Api`, `RetriesExhausted`, `InvalidSseFrame`, `BackoffOverflow`
  - Each variant carries context: `status`, `request_id`, `body`, `retryable` flag
  - Implements `Display`, `std::error::Error`, `From<reqwest::Error>`, `From<std::io::Error>`, `From<serde_json::Error>`, `From<VarError>`
  - Classifying methods: `is_retryable()`, `request_id()`, `safe_failure_class()`, `is_context_window_failure()`
- `conversation::RuntimeError` -- lightweight string-based error with `new(message: impl Into<String>)`
- `conversation::ToolError` -- lightweight string-based error for tool invocation failures

**Result Propagation:**
- Use `?` operator throughout
- `From` impls on error types handle automatic conversion
- Pattern: `impl From<reqwest::Error> for ApiError` so HTTP code can just `?`

**Constructor Pattern for Errors:**
- Named constructors with context: `ApiError::json_deserialize(provider, model, body, source)`
- Builder-style: `ApiError::missing_credentials_with_hint(provider, env_vars, hint)`

## Async Patterns

**Runtime:** tokio with `rt-multi-thread` feature

**Usage Patterns:**
- `#[tokio::test]` for async integration tests
- `tokio::sync::Mutex` for async-safe shared state (NOT `std::sync::Mutex` in async contexts)
- `tokio::spawn` for background tasks
- `tokio::select!` for shutdown-signal handling (e.g., `crates/mock-anthropic-service/src/lib.rs`)
- Manual `tokio::runtime::Runtime::new()` in sync test contexts (e.g., `crates/rusty-claude-cli/tests/mock_parity_harness.rs`)
- `oneshot::channel` for shutdown signaling
- `Arc<Mutex<Vec<T>>>` for captured request state in test servers

**Blocking in Async:**
- `reqwest::blocking::Client` used in `tools` crate (separate from async API client)
- `std::process::Command` used synchronously for subprocess execution

## Trait Design Patterns

**Traits in the codebase:**

| Trait | Location | Purpose |
|-------|----------|---------|
| `ApiClient` | `crates/runtime/src/conversation.rs:53` | Streaming API contract: `fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>` |
| `ToolExecutor` | `crates/runtime/src/conversation.rs:58` | Tool dispatch: `fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError>` |
| `TelemetrySink` | `crates/telemetry/src/lib.rs:205` | Event recording: `fn record(&self, event: TelemetryEvent)` -- requires `Send + Sync` |
| `PermissionPrompter` | `crates/runtime/src/permissions.rs:86` | Interactive permission prompting |
| `PluginLifecycle` | `crates/runtime/src/plugin_lifecycle.rs:214` | Plugin discovery and health lifecycle |
| `Plugin` | `crates/plugins/src/lib.rs:410` | Plugin tool definition contract |
| `Provider` | `crates/api/src/providers/mod.rs:17` | Provider-agnostic model client interface |
| `HookProgressReporter` | `crates/runtime/src/hooks.rs:58` | Hook lifecycle event reporting |

**Trait Design Conventions:**
- Object-safe traits (no `async` in trait methods -- use `&mut self` not `self`)
- Traits require `Send + Sync` when needed for cross-thread use
- Implementations stored as `Box<dyn Trait>` or `Arc<dyn Trait>`
- `StaticToolExecutor` as a test-friendly impl using `FnMut` closures

## Serde Patterns

**Serialization:**
- `#[derive(Serialize, Deserialize)]` on all wire-format types
- `#[serde(tag = "type", rename_all = "snake_case")]` on enums that need discriminated union JSON (e.g., `InputContentBlock`, `OutputContentBlock`, `StreamEvent`, `TelemetryEvent`)
- `#[serde(skip_serializing_if = "Option::is_none")]` on optional fields to omit nulls
- `#[serde(skip_serializing_if = "Vec::is_empty")]` on optional collections
- `#[serde(skip_serializing_if = "Map::is_empty")]` on optional maps
- `#[serde(default)]` on fields that should use `Default::default()` when missing
- `#[serde(rename = "type")]` to map Rust `kind` field to JSON `"type"` key
- `#[serde(rename = "kebab-case")]` for enum variants that need kebab-case (e.g., `FilesystemIsolationMode`)
- `#[serde(rename = "PreToolUse")]` preserving PascalCase hook names from upstream

**Deserialization:**
- `serde_json::from_str::<T>()` for parsing
- `serde_json::Value` used as intermediate for unstructured data (tool input schemas, config values)
- Custom `JsonValue` type in `crates/runtime/src/json.rs` for config handling

## Public API Design

**Visibility Hierarchy:**
- `pub` -- exported from crate for workspace consumers
- `pub(crate)` -- visible within crate but not re-exported (rare, used for test helpers)
- `pub use` in `lib.rs` -- the canonical public surface of each crate
- Private `mod foo;` -- implementation details not accessible outside the crate

**Re-export Strategy:**
- Every public item goes through `pub use` in `lib.rs`
- Selective: only types/functions needed by other crates are re-exported
- Pattern: `pub use config::{ConfigEntry, ConfigLoader, ...};` lists all public names

**`#[must_use]` Annotations:**
- Extensively used: 496 occurrences across 59 files
- Applied to: constructors (`new()`, `with_*()`), pure functions, query methods
- Example: `pub fn new(entries: Vec<ToolManifestEntry>) -> Self` has `#[must_use]`
- Every pure/computational function returns `#[must_use]`

## Module Visibility and Re-export Patterns

**Pattern:**
```
// lib.rs
mod internal_module;        // private -- not accessible from outside
pub mod public_module;      // public -- directly accessible

pub use internal_module::{SpecificType, specific_fn};  // cherry-pick re-exports
```

**Notable examples:**
- `crates/runtime/src/lib.rs`: 50+ private modules, selective `pub use` for ~170 public items
- `crates/api/src/lib.rs`: 6 private modules, selective re-exports
- `crates/plugins/src/lib.rs`: Private `mod hooks`, `pub use hooks::{HookEvent, HookRunResult, HookRunner}`

**Conditionally public modules:**
- `#[cfg(test)] pub mod test_isolation;` -- only compiled and public during test builds

## Configuration Patterns

**Multi-layer config loading** (`crates/runtime/src/config.rs`):
- Three config scopes with precedence: `Local > Project > User`
- `ConfigSource` enum: `User`, `Project`, `Local`
- `ConfigEntry` pairs a source with a file path
- `RuntimeConfig` holds merged `BTreeMap<String, JsonValue>` plus structured feature views
- Feature-specific config structs: `RuntimeFeatureConfig`, `RuntimeHookConfig`, `RuntimePluginConfig`, `McpConfigCollection`
- `CLAW_CONFIG_HOME` env var overrides the config directory
- Config validation with diagnostics in `crates/runtime/src/config_validate.rs`

**Serde-based deserialization from JSON config files** -- not a typed config file format.

## Logging/Tracing Patterns

**No `tracing` or `log` crate usage.** The codebase does not use the `tracing` or `log` crates.

**Diagnostic output:**
- `eprintln!()` for error messages to stderr (primarily in `crates/rusty-claude-cli/src/main.rs` and `crates/tools/src/lib.rs`)
- Custom `TelemetrySink` trait for structured event recording (`crates/telemetry/src/lib.rs`)
- `MemoryTelemetrySink` for test/inspection, `JsonlTelemetrySink` for file persistence
- `SessionTracer` wraps a `TelemetrySink` with session-scoped sequencing

**Pattern:** Structured events over string logging. Telemetry events carry typed fields.

## Comment/Doc Style

**Module-level docs:**
- `//!` doc comments on `lib.rs` and key modules
- Example: `crates/runtime/src/lib.rs`: `//! Core runtime primitives for the \`claw\` CLI and supporting crates.`

**Item docs:**
- `///` on public types, functions, constants
- Doc comments explain "what" and "why", not "how"
- Example: `/// Fully merged runtime configuration plus parsed feature-specific views.`

**Inline comments:**
- Used for non-obvious logic, not for restating code
- TODO/FIXME tracking inline (not centralized)

**Test comments:**
- `// given / // when / // then` pattern for clarity (e.g., `crates/api/src/error.rs:522`)
- Test function names are descriptive sentences: `json_deserialize_error_includes_provider_model_and_truncated_body_snippet`

## Function Design

**Size:**
- Functions tend to be focused; large functions annotated with `#[allow(clippy::too_many_lines)]` where unavoidable
- Builder methods kept short (2-5 lines)

**Parameters:**
- `impl Into<String>` for string parameters that accept `&str` or `String`
- `impl AsRef<Path>` for path parameters
- Struct parameters for >3 related values (e.g., `CompactionConfig`, `SandboxDetectionInputs`)

**Return Values:**
- `Result<T, DomainError>` for fallible operations
- `#[must_use]` on pure computations
- `Option<T>` for truly optional returns
- Tuple structs for compound returns where a named struct would be overkill (rare)

## Anti-patterns and Inconsistencies

**Giant monolithic files:**
- `crates/rusty-claude-cli/src/main.rs` is 11,788 lines -- the entire CLI application in one file
- `crates/tools/src/lib.rs` is 9,682 lines -- all tool implementations in one file
- `crates/commands/src/lib.rs` is 5,634 lines -- all slash commands in one file
- These should be split into submodules

**Error type inconsistency:**
- `ApiError` is rich and structured; `RuntimeError` and `ToolError` are plain string wrappers
- No `thiserror` or `anyhow` -- all error Display/Error impls are hand-written

**Mixed blocking/async:**
- `tools` crate uses `reqwest::blocking::Client` while `api` crate uses async reqwest
- `std::process::Command` used synchronously even in async contexts

**No centralized test utilities crate:**
- Each integration test file duplicates `env_lock()`, `EnvVarGuard`, `unique_temp_dir()`, `TEMP_COUNTER` patterns
- The `EnvVarGuard` is redefined in `crates/api/tests/proxy_integration.rs` and `crates/api/tests/provider_client_integration.rs` independently

**Global mutable state via `OnceLock`:**
- Multiple `global_*()` functions in `crates/tools/src/lib.rs` use `OnceLock` for singletons (LSP registry, MCP registry, etc.)
- This pattern is pragmatic but makes testing harder

---

*Convention analysis: 2026-04-14*
