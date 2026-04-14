# Testing Patterns

**Analysis Date:** 2026-04-14

## Test Framework

**Runner:**
- `cargo test --workspace` (Rust built-in test harness, libtest)
- No additional test framework (no proptest, no criterion)

**Run Commands:**
```bash
cargo test --workspace                       # Run all tests
cargo test --workspace --all-targets         # Include binaries
cargo test -p <crate-name>                   # Single crate
cargo test -p api --test client_integration  # Single integration test file
cargo test -- --nocapture                    # Show stdout (used by parity harness)
cargo test <test_name>                       # Single test by name filter
```

**Lint/Format Verification:**
```bash
cargo fmt                                    # Format check
cargo clippy --workspace --all-targets -- -D warnings  # Lint
```

**Parity Harness Script:**
```bash
./scripts/run_mock_parity_harness.sh         # Runs mock parity harness with --nocapture
```

## Test File Organization

**Two patterns coexist:**

1. **`#[cfg(test)] mod tests` in source files** -- Unit tests colocated with implementation
   - Used by every crate for unit-level tests
   - Found at the bottom of source files (after production code)
   - Example: `crates/runtime/src/conversation.rs` line 822, `crates/api/src/error.rs` line 396

2. **`tests/` directory per crate** -- Integration tests that compile against the crate's public API
   - `crates/api/tests/client_integration.rs` -- API client HTTP tests
   - `crates/api/tests/openai_compat_integration.rs` -- OpenAI-compatible provider tests
   - `crates/api/tests/proxy_integration.rs` -- Proxy configuration tests
   - `crates/api/tests/provider_client_integration.rs` -- Provider routing tests
   - `crates/runtime/tests/integration_tests.rs` -- Cross-module runtime wiring tests
   - `crates/rusty-claude-cli/tests/mock_parity_harness.rs` -- End-to-end CLI parity tests
   - `crates/rusty-claude-cli/tests/cli_flags_and_config_defaults.rs` -- CLI flag tests
   - `crates/rusty-claude-cli/tests/output_format_contract.rs` -- JSON output format tests
   - `crates/rusty-claude-cli/tests/compact_output.rs` -- Compact mode output tests
   - `crates/rusty-claude-cli/tests/resume_slash_commands.rs` -- Session resume tests

## Test Structure

**Unit test pattern** (colocated `#[cfg(test)] mod tests`):
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn descriptive_sentence_describing_behavior() {
        // given
        let config = CompactionConfig::default();

        // when
        let result = should_compact(&session, config);

        // then
        assert!(result);
    }
}
```

**Integration test pattern** (tests/ directory):
```rust
use api::{ApiClient, ApiError, ...};
use serde_json::json;

#[tokio::test]
async fn send_message_posts_json_and_parses_response() {
    let server = spawn_server(state.clone(), responses).await;
    let client = ApiClient::new("test-key")
        .with_base_url(server.base_url());
    let response = client.send_message(&request).await.expect("request should succeed");
    assert_eq!(response.id, "msg_test");
}
```

**CLI integration test pattern** (subprocess-based):
```rust
#[test]
fn status_command_applies_model_and_permission_mode_flags() {
    let temp_dir = unique_temp_dir("status-flags");
    fs::create_dir_all(&temp_dir).expect("temp dir should exist");

    let output = Command::new(env!("CARGO_BIN_EXE_claw"))
        .current_dir(&temp_dir)
        .args(["--model", "sonnet", "--permission-mode", "read-only", "status"])
        .output()
        .expect("claw should launch");

    assert_success(&output);
    let stdout = String::from_utf8(output.stdout).expect("stdout should be utf8");
    assert!(stdout.contains("Model            claude-sonnet-4-6"));
}
```

## Test Isolation

**Environment Lock Pattern:**
Tests that modify env vars use a static `Mutex` lock to prevent parallel interference:
```rust
fn env_lock() -> std::sync::MutexGuard<'static, ()> {
    static LOCK: OnceLock<StdMutex<()>> = OnceLock::new();
    LOCK.get_or_init(|| StdMutex::new(()))
        .lock()
        .unwrap_or_else(std::sync::PoisonError::into_inner)
}
```
Used in: `crates/api/tests/client_integration.rs`, `crates/api/tests/proxy_integration.rs`, `crates/api/tests/provider_client_integration.rs`

**`EnvVarGuard` pattern** (RAII env var save/restore):
```rust
struct EnvVarGuard { key: &'static str, original: Option<OsString> }
impl EnvVarGuard {
    fn set(key: &'static str, value: Option<&str>) -> Self { /* ... */ }
}
impl Drop for EnvVarGuard {
    fn drop(&mut self) { /* restore original value */ }
}
```
Defined independently in multiple test files -- not shared.

**`EnvLock` from plugins test_isolation** (`crates/plugins/src/test_isolation.rs`):
- RAII guard that creates an isolated temp home, sets `HOME`/`XDG_CONFIG_HOME`/`XDG_DATA_HOME`
- Cleans up temp dir on drop
- Mutex-guarded for serial access

**`test_env_lock()` from runtime** (`crates/runtime/src/lib.rs`):
- Shared across runtime unit tests to serialize env-dependent tests
- Uses `PoisonError::into_inner` to tolerate poisoned mutex (from test panics)

**Temporary directories:**
- `unique_temp_dir(prefix)` pattern with `AtomicU64` counter used in CLI tests
- Creates per-test directories under `std::env::temp_dir()` with cleanup in test teardown

## Mocking

**No mocking framework** -- all mocks are hand-built.

**Mock HTTP servers** (most significant mock pattern):
- Inline TCP servers using `tokio::net::TcpListener` in integration tests
- `spawn_server()` helper spawns a tokio task that accepts connections and responds with pre-canned HTTP responses
- `CapturedRequest` struct records method, path, headers, body, and scenario tag from each request

**`MockAnthropicService`** (`crates/mock-anthropic-service/src/lib.rs`):
- Full mock Anthropic API server bound to `127.0.0.1:0` (random port)
- Scenario-based: parses `PARITY_SCENARIO:` prefix from system prompt to select mock behavior
- 12 hardcoded scenarios (e.g., `StreamingText`, `ReadFileRoundtrip`, `WriteFileDenied`)
- Records all requests in `Arc<Mutex<Vec<CapturedRequest>>>` for post-test assertions
- Shutdown via `oneshot::channel` + `JoinHandle::abort()`
- Also available as standalone binary: `cargo run -p mock-anthropic-service -- --bind 127.0.0.1:0`

**`StaticToolExecutor`** (`crates/runtime/src/conversation.rs:800`):
- Test double for `ToolExecutor` trait
- `register(name, |input| Ok(result))` builder pattern for scripted tool responses

**`ScriptedApiClient`** (`crates/runtime/src/conversation.rs`):
- Test double for `ApiClient` trait
- Returns pre-scripted `AssistantEvent` sequences per call count

**`MemoryTelemetrySink`** (`crates/telemetry/src/lib.rs:210`):
- In-memory `TelemetrySink` that records events to `Mutex<Vec<TelemetryEvent>>`
- Used to verify telemetry output in tests

## The Mock Parity Harness

**Purpose:** End-to-end validation that the Rust CLI binary produces correct behavior across 12 scripted scenarios, driven by a mock Anthropic API server.

**Entry point:** `crates/rusty-claude-cli/tests/mock_parity_harness.rs`

**Scenario manifest:** `/Users/zhangnaiqi/workspace/claw-code/rust/mock_parity_scenarios.json`
- 12 scenarios across categories: `baseline`, `file-tools`, `permissions`, `multi-tool-turns`, `bash`, `plugin-paths`, `session-compaction`, `token-usage`

**How it works:**
1. Spawns `MockAnthropicService` on a random port
2. For each scenario:
   a. Creates an isolated temp workspace with config-home and home directories
   b. Runs `prepare` callback to set up fixtures (e.g., write fixture files)
   c. Launches `claw` binary as a subprocess with `env_clear()` and controlled env vars
   d. Captures stdout/stderr and parses JSON output
   e. Runs `assert` callback to verify expectations (file contents, exit code, output text)
   f. Cleans up temp directory
3. Verifies total captured `/v1/messages` request count matches expected (21 requests for 12 scenarios)
4. Optionally writes scenario reports

**Scenario categories covered:**

| Category | Scenarios |
|----------|-----------|
| baseline | `streaming_text` |
| file-tools | `read_file_roundtrip`, `grep_chunk_assembly`, `write_file_allowed` |
| permissions | `write_file_denied`, `bash_permission_prompt_approved`, `bash_permission_prompt_denied` |
| multi-tool-turns | `multi_tool_turn_roundtrip` |
| bash | `bash_stdout_roundtrip` |
| plugin-paths | `plugin_tool_roundtrip` |
| session-compaction | `auto_compact_triggered` |
| token-usage | `token_cost_reporting` |

**Run:** `./scripts/run_mock_parity_harness.sh` or `cargo test -p rusty-claude-cli --test mock_parity_harness -- --nocapture`

## Test Data Patterns

**Scenario JSON manifest** (`mock_parity_scenarios.json`):
- Name, category, description, parity_refs for each scenario
- Tests validate that harness cases stay aligned with manifest entries

**Temp workspace fixtures:**
- Created per-test with `unique_temp_dir()` + `fs::create_dir_all()`
- Fixture files written directly: `fs::write(workspace.join("fixture.txt"), "content")`
- Plugin fixtures: full directory trees with `plugin.json` manifests

**Hardcoded JSON in tests:**
- API response bodies constructed inline as `concat!()` string literals
- Request assertions parse `serde_json::Value` from captured request body
- Example: `"\"id\":\"msg_test\",...\"content\":[{\"type\":\"text\",\"text\":\"Hello from Claude\"}]"`

**No golden files or snapshot testing.**

## Coverage of Core Flows

**Well-tested:**
- **API client HTTP layer**: 14 `#[tokio::test]` tests in `crates/api/tests/client_integration.rs` covering send_message, streaming, context window preflight, telemetry, retries, auth headers
- **OpenAI-compatible provider**: 6 async tests in `crates/api/tests/openai_compat_integration.rs`
- **Provider routing**: 4 tests in `crates/api/tests/provider_client_integration.rs` (xAI, Anthropic, explicit auth)
- **Proxy configuration**: 7 tests in `crates/api/tests/proxy_integration.rs`
- **Error classification**: 8 unit tests in `crates/api/src/error.rs` covering failure classes, truncation, hints
- **SSE parsing**: 8 unit tests in `crates/api/src/sse.rs`
- **Prompt cache**: 8 unit tests in `crates/api/src/prompt_cache.rs`
- **Anthropic provider internals**: 24 unit tests in `crates/api/src/providers/anthropic.rs`
- **Provider resolution/aliasing**: 25 unit tests in `crates/api/src/providers/mod.rs`
- **OpenAI compat parsing**: 25 unit tests in `crates/api/src/providers/openai_compat.rs`
- **Session persistence**: Tests in `crates/runtime/src/session.rs` (unit tests at line 1143, JSON roundtrip at 1503)
- **Config loading and validation**: Tests in `crates/runtime/src/config.rs` (line 1244) and `crates/runtime/src/config_validate.rs` (line 534)
- **Permissions**: Tests in `crates/runtime/src/permissions.rs` (line 471) and `crates/runtime/src/permission_enforcer.rs` (line 274)
- **Policy engine**: 8 tests in `crates/runtime/src/policy_engine.rs` (line 218)
- **Stale branch detection**: 16 tests in `crates/runtime/src/stale_branch.rs` (line 169)
- **Green contract**: Tests in `crates/runtime/src/green_contract.rs` (line 81)
- **Lane events deduplication**: 7 tests in `crates/runtime/src/lane_events.rs` (line 252)
- **Runtime cross-module wiring**: 11 tests in `crates/runtime/tests/integration_tests.rs`
- **Hooks**: Tests in `crates/runtime/src/hooks.rs` (line 819) and `crates/plugins/src/hooks.rs` (line 367)
- **Plugins**: Tests in `crates/plugins/src/lib.rs` (line 2287)
- **CLI flag parsing**: Tests in `crates/rusty-claude-cli/tests/cli_flags_and_config_defaults.rs`
- **JSON output format**: Tests in `crates/rusty-claude-cli/tests/output_format_contract.rs`
- **Compact output**: Tests in `crates/rusty-claude-cli/tests/compact_output.rs`
- **Session resume**: Tests in `crates/rusty-claude-cli/tests/resume_slash_commands.rs`
- **Slash commands**: 43 unit tests in `crates/commands/src/lib.rs` (line 4117)
- **Bash validation**: Tests in `crates/runtime/src/bash_validation.rs` (line 711)
- **Telemetry**: 3 tests in `crates/telemetry/src/lib.rs` (line 430)
- **Conversation runtime tool loop**: Tests in `crates/runtime/src/conversation.rs` (line 822)
- **Compaction**: Tests in `crates/runtime/src/compact.rs` (line 554)
- **Recovery recipes**: Tests in `crates/runtime/src/recovery_recipes.rs` (line 308)
- **MCP lifecycle**: Tests in `crates/runtime/src/mcp_lifecycle_hardened.rs` (line 391) and `crates/runtime/src/mcp_server.rs` (line 300)

**Moderately tested:**
- **Worker boot**: Tests in `crates/runtime/src/worker_boot.rs` (line 879)
- **OAuth flow**: Tests in `crates/runtime/src/oauth.rs` (line 463)
- **Branch lock collision**: Tests in `crates/runtime/src/branch_lock.rs` (line 79)
- **Task packet validation**: Tests in `crates/runtime/src/task_packet.rs` (line 92)

**Gaps:**
- **MCP stdio transport**: Limited test coverage for actual stdio process lifecycle (only unit tests at `crates/runtime/src/mcp_stdio.rs` line 21 and 1408)
- **MCP client bootstrap**: Minimal tests at `crates/runtime/src/mcp_client.rs` line 132
- **MCP tool bridge**: Minimal tests at `crates/runtime/src/mcp_tool_bridge.rs` line 313
- **Remote session**: Tests at `crates/runtime/src/remote.rs` line 253 -- limited coverage of proxy/ws flows
- **LSP client**: Tests at `crates/runtime/src/lsp_client.rs` line 299 -- limited
- **Sandbox detection**: Basic tests at `crates/runtime/src/sandbox.rs` line 306
- **Git context**: Tests at `crates/runtime/src/git_context.rs` line 147
- **Tool execution (tools crate)**: Large file (9682 lines) with tests at line 6115 but likely incomplete for all tool paths
- **Conversation runtime edge cases**: No tests for concurrent turns, cancellation mid-stream
- **PDF extraction**: Tests at `crates/tools/src/pdf_extract.rs` line 308 but PDF parsing is inherently fragile
- **Lane completion**: Tests at `crates/tools/src/lane_completion.rs` line 92

## Flaky Test Mitigations

**Env var serialization:**
- `env_lock()` pattern prevents concurrent tests from reading/writing the same env vars
- `PoisonError::into_inner()` on mutex guards tolerates panics without cascading failures

**Port conflicts:**
- Mock servers bind to `127.0.0.1:0` (OS-assigned random port) -- no port collision risk

**Temp directory isolation:**
- `unique_temp_dir()` with atomic counter avoids path collisions
- Explicit cleanup in test teardown: `fs::remove_dir_all()`

**Process cleanup:**
- `MockAnthropicService::Drop` sends shutdown signal and aborts join handle

**CI-specific guard:**
- Recent commits mention "Keep poisoned test locks from cascading across unrelated regressions" (commit `f91d156`) and "Keep local clawhip artifacts from tripping routine repo work" (commit `6b4bb4a`)

## Running Subsets of Tests

```bash
# All unit tests in a crate
cargo test -p runtime

# Specific integration test file
cargo test -p api --test client_integration

# Tests matching a name pattern
cargo test -p runtime config_

# Async tests only (all tokio::test)
cargo test -p api

# CLI binary tests
cargo test -p rusty-claude-cli

# Mock parity harness (end-to-end)
cargo test -p rusty-claude-cli --test mock_parity_harness -- --nocapture

# Skip slow CLI tests (run only fast unit tests)
cargo test -p api -p runtime -p telemetry -p plugins -p commands

# Run with output visible
cargo test -p runtime stale_branch -- --nocapture
```

## Test Helpers and Utilities

**Duplicated across test files (not shared):**

| Helper | Found In |
|--------|----------|
| `env_lock()` | `crates/api/tests/client_integration.rs`, `crates/api/tests/proxy_integration.rs`, `crates/api/tests/provider_client_integration.rs` |
| `EnvVarGuard` | `crates/api/tests/proxy_integration.rs`, `crates/api/tests/provider_client_integration.rs` (separate copies) |
| `unique_temp_dir()` | `crates/rusty-claude-cli/tests/cli_flags_and_config_defaults.rs`, `crates/rusty-claude-cli/tests/output_format_contract.rs`, `crates/rusty-claude-cli/tests/resume_slash_commands.rs`, `crates/rusty-claude-cli/tests/compact_output.rs` |
| `assert_success()` | `crates/rusty-claude-cli/tests/cli_flags_and_config_defaults.rs`, `crates/rusty-claude-cli/tests/resume_slash_commands.rs` |
| `write_session()` | `crates/rusty-claude-cli/tests/cli_flags_and_config_defaults.rs`, `crates/rusty-claude-cli/tests/resume_slash_commands.rs` |
| `workspace_session()` | `crates/rusty-claude-cli/tests/resume_slash_commands.rs` |
| `run_claw()` | `crates/rusty-claude-cli/tests/compact_output.rs`, `crates/rusty-claude-cli/tests/resume_slash_commands.rs` |
| `test_env_lock()` | `crates/runtime/src/lib.rs` |
| `EnvLock` | `crates/plugins/src/test_isolation.rs` |
| `MockAnthropicService` | `crates/mock-anthropic-service/src/lib.rs` (shared via dev-dependency) |

**Shared via dev-dependencies:**
- `mock-anthropic-service` is a `dev-dependency` of `rusty-claude-cli`
- `MemoryTelemetrySink` from `telemetry` crate used in tests across crates

**No shared test utility crate exists.** Helper duplication is the most significant test infrastructure gap.

---

*Testing analysis: 2026-04-14*
