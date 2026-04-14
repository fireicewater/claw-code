# Technology Stack

**Analysis Date:** 2026-04-14

## Languages

**Primary:**
- Rust 2021 edition - All crates in the workspace

**Secondary:**
- Shell (POSIX) - Build script in `rusty-claude-cli/build.rs`, subprocess execution in `runtime/src/bash.rs`
- JSON - Wire format for all API interactions, configuration, session persistence

## Runtime

**Environment:**
- Rust edition 2021, no specific MSRV pinned (follows stable Rust)
- Target: native binary (`claw`)

**Package Manager:**
- Cargo
- Lockfile: `Cargo.lock` (present, 54 KB)

## Workspace Structure

**Resolver:** Edition 2024 resolver (v2)

**Workspace root:** `rust/Cargo.toml`

**Members (9 crates):**

| Crate | Type | Purpose |
|-------|------|---------|
| `api` | library | HTTP clients for Anthropic/OpenAI-compatible APIs, SSE parsing, types |
| `runtime` | library | Session management, config, MCP, sandbox, permissions, bash execution |
| `tools` | library | Tool dispatch and execution (file ops, search, web fetch, PDF) |
| `commands` | library | Slash command registry and handlers |
| `plugins` | library | Plugin system (discovery, hooks, lifecycle) |
| `telemetry` | library | Analytics events, session tracing, JSONL telemetry sink |
| `compat-harness` | library | Upstream compatibility manifest extraction and bootstrap plan |
| `mock-anthropic-service` | binary | Local mock Anthropic API server for testing |
| `rusty-claude-cli` | binary | CLI entrypoint (`claw` binary) |

**Build script:** `crates/rusty-claude-cli/build.rs` -- injects `GIT_SHA`, `TARGET`, and `BUILD_DATE` at compile time via `cargo:rustc-env`.

## Key Dependencies

### Workspace-shared
- `serde_json = "1"` -- JSON serialization/deserialization across all crates

### api crate
- `reqwest 0.12` (features: `json`, `rustls-tls`) -- HTTP client, TLS via rustls (no OpenSSL dependency)
- `tokio 1` (features: `io-util`, `macros`, `net`, `rt-multi-thread`, `time`) -- Async runtime
- `runtime` (path) -- Re-exports config, OAuth, pricing types
- `telemetry` (path) -- Request profiling, session tracing

### runtime crate
- `tokio 1` (features: `io-std`, `io-util`, `macros`, `process`, `rt`, `rt-multi-thread`, `time`) -- Process spawning, async IO
- `sha2 0.10` -- OAuth PKCE challenge hashing
- `glob 0.3` -- Glob pattern matching for file search
- `regex 1` -- Pattern matching (grep search)
- `walkdir 2` -- Recursive directory traversal
- `serde 1` (features: `derive`) -- Serialization
- `plugins` (path) -- Plugin hooks, registry
- `telemetry` (path) -- Telemetry events

### rusty-claude-cli crate
- `rustyline 15` -- Readline-style line editing for REPL input
- `crossterm 0.28` -- Terminal control (colors, cursor, clearing) for TUI rendering
- `pulldown-cmark 0.13` -- Markdown rendering (streaming parser for assistant responses)
- `syntect 5` -- Syntax highlighting for code blocks in terminal output
- `tokio 1` (features: `rt-multi-thread`, `signal`, `time`) -- Async runtime for API calls
- `api`, `runtime`, `tools`, `commands`, `plugins`, `compat-harness` (path deps) -- All other crates

### tools crate
- `reqwest 0.12` (features: `blocking`, `rustls-tls`) -- Synchronous HTTP client for web fetch tool
- `flate2 1` -- PDF stream decompression (`pdf_extract.rs`)
- `tokio 1` (features: `rt-multi-thread`) -- Async runtime

### plugins crate
- `serde 1` (features: `derive`) -- Plugin manifest serialization
- `serde_json` (workspace) -- JSON parsing for plugin configs

### telemetry crate
- `serde 1` (features: `derive`) -- Event serialization

## Configuration

**Environment:**
- Config loaded from three layers (precedence order):
  1. `$CLAW_CONFIG_HOME/settings.json` (user scope)
  2. `$HOME/.claw/settings.json` (user scope, fallback)
  3. `.claw/settings.json` in CWD (project scope)
- `.env` file in CWD is parsed for credential discovery (minimal KEY=VALUE grammar)
- `CLAW_CONFIG_HOME` env var overrides the home config directory

**Key environment variables:**
- `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_BASE_URL` -- Anthropic provider
- `OPENAI_API_KEY`, `OPENAI_BASE_URL` -- OpenAI-compatible provider
- `XAI_API_KEY`, `XAI_BASE_URL` -- xAI provider
- `DASHSCOPE_API_KEY`, `DASHSCOPE_BASE_URL` -- Alibaba DashScope provider
- `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` (and lowercase variants) -- HTTP proxy
- `CLAW_CONFIG_HOME` -- Config directory override

**Build:**
- No custom profiles defined (uses Cargo defaults: `dev` and `release`)
- No feature flags defined on any crate
- `build.rs` only in `rusty-claude-cli` (git SHA, target, build date injection)

## Key Rust Patterns

**Error handling:**
- Custom enum-based errors (e.g., `ApiError` in `api/src/error.rs` with 12 variants)
- `thiserror` is NOT used; manual `Display` and `Error` implementations
- `From` conversions for `reqwest::Error`, `std::io::Error`, `serde_json::Error`, `std::env::VarError`
- Error enrichment (e.g., `enrich_bearer_auth_error` adds hints for common misconfiguration)
- `is_retryable()` classification on errors for automatic retry logic

**Async patterns:**
- Tokio multi-threaded runtime throughout
- `async/await` with `Box::pin` for trait object futures (`ProviderFuture`)
- `tokio::process::Command` for subprocess spawning in `runtime`
- `reqwest::blocking::Client` in `tools` crate for synchronous web fetch

**Traits:**
- `Provider` trait (`api/src/providers/mod.rs`) -- `send_message` and `stream_message` methods
- `ToolExecutor` trait -- Dispatches tool calls by name
- `TelemetrySink` trait -- Consumes analytics events
- `StringExt` private trait in openai_compat -- `if_empty_then` helper

**Serialization:**
- `serde` derive macros on all data types
- `#[serde(tag = "type", rename_all = "snake_case")]` for tagged enums
- `#[serde(skip_serializing_if = "Option::is_none")]` and `#[serde(default)]` extensively
- Untagged enums for flexible JSON deserialization (e.g., `JsonRpcId`)

**Concurrency:**
- `Arc<Mutex<T>>` for shared state (e.g., `PromptCache`, `McpToolRegistry`)
- `OnceLock<T>` for global singletons (`LspRegistry`, `McpToolRegistry`, `TeamRegistry`, `CronRegistry`)
- `AtomicU64` for monotonic counters (jitter generation, plugin invocation IDs)
- Poisoned mutex locks are unwrapped with `PoisonError::into_inner` (non-fatal)

**Builder pattern:**
- Extensive use of `with_*` methods for configuration (e.g., `AnthropicClient::new(key).with_base_url(url).with_retry_policy(...)`)
- `#[must_use]` on all builder methods

## Lint Configuration

**Workspace lints** (enforced on all crates via `[workspace.lints]`):

**Rust lints:**
- `unsafe_code = "forbid"` -- No unsafe code allowed anywhere in the workspace

**Clippy lints:**
- `all = "warn"` -- All default Clippy lints are warnings
- `pedantic = "warn"` -- Strict pedantic lints enabled
- `module_name_repetitions = "allow"` -- Allows module-prefixed names
- `missing_panics_doc = "allow"` -- No documentation required for panics
- `missing_errors_doc = "allow"` -- No documentation required for errors

**Verification commands** (from `CLAUDE.md`):
```bash
cargo fmt
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

## Unsafe Usage Policy

**`unsafe_code` is forbidden at the workspace level.** The `[workspace.lints.rust]` table sets `unsafe_code = "forbid"`, meaning no crate can use `unsafe` blocks. This is enforced by the compiler.

---

*Stack analysis: 2026-04-14*
