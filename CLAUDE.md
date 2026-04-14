# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Claw Code is a public Rust implementation of the `claw` CLI agent harness — an interactive REPL and one-shot prompt runner that talks to Anthropic/OpenAI-compatible APIs with built-in tool execution. The canonical code lives in `rust/`.

## Build & verification

All commands run from `rust/`:

```bash
cargo build --workspace
cargo fmt
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo test --workspace -p rusty-claude-cli  # CLI integration tests only
cargo test -p rusty-claude-cli test_name       # single test by name
cargo run -p rusty-claude-cli -- --help        # live CLI help
```

`unsafe_code` is forbidden at the workspace level. Clippy pedantic is enabled with a few allowed exceptions (see `Cargo.toml` lints).

## Workspace layout (rust/)

9 crates with a clear dependency hierarchy:

```
rusty-claude-cli  (binary: `claw`)
  ├── api          — provider clients, SSE streaming, auth, request preflight
  ├── commands     — slash command registry, parsing, help rendering
  ├── tools        — 40 tool specs + execution (Bash, Read, Write, Edit, Grep, Glob, Web, Agent, Todo, etc.)
  ├── runtime      — core runtime: session, config, permissions, MCP, prompt assembly, conversation loop
  │   ├── plugins  — plugin metadata, lifecycle, install/enable/disable
  │   └── telemetry— session tracing types
  ├── compat-harness       — extracts manifests from upstream TS source
  └── mock-anthropic-service — deterministic mock for parity tests
```

Key dependency direction: `cli → api + tools + commands`, `tools → runtime`, `runtime` is the foundation layer.

## Architecture

**ConversationRuntime** (`runtime/src/conversation.rs`) is the core loop. It manages the turn cycle: build prompt → call API → receive assistant events → dispatch tool calls → feed results back → repeat until done.

**Tool execution** flows through `tools::execute_tool()` which dispatches to concrete `run_*` handlers. Tool specs are defined in `tools::mvp_tool_specs()`. Runtime state (tasks, MCP, LSP, teams, cron) uses process-wide `OnceLock` singletons accessed via `global_*_registry()`.

**Config loading** (`runtime/src/config.rs`): `ConfigLoader::discover()` merges settings with precedence user > project > local, reading `.claw.json` / `.claude.json` / `.claude/settings.json`.

**Permission enforcement** (`runtime/src/permission_enforcer.rs`): layers on top of `PermissionPolicy` to gate file writes, bash commands, and tool access by permission mode.

**Mock parity harness**: `scripts/run_mock_parity_harness.sh` runs deterministic end-to-end CLI tests against a local mock Anthropic service. Scenarios tracked in `mock_parity_scenarios.json`.

## Repository shape

- `rust/` — Rust workspace and the `claw` binary (the entire runtime)

## Working agreement

- Prefer small, reviewable changes.
- Keep shared defaults in `.claude.json`; reserve `.claude/settings.local.json` for machine-local overrides.
- Do not overwrite existing `CLAUDE.md` content automatically; update it intentionally.
- Default model: `claude-opus-4-6`. Model aliases: `opus`, `sonnet`, `haiku`.
- The CLI surface moves quickly — always verify with `cargo run -p rusty-claude-cli -- --help` rather than relying on documentation.
