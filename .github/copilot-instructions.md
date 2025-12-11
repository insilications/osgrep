# osgrep Copilot Instructions

## Architecture & Ownership
- Workspace layout: CLI binary [cli/src](../cli/src), Node N-API addon [native/src](../native/src), MCP server [mcp/src/main.rs](../mcp/src/main.rs), Claude hooks [plugins/osgrep](../plugins/osgrep), and Vitest suites under [tests](../tests).
- CLI entrypoint plus subcommand handling live in [cli/src/main.rs](../cli/src/main.rs); add flags by extending the `Commands` enum, then wiring a `cmd_*` helper.

## Indexing & Search Flow
- `cmd_index` orchestrates collection (via `collect_files`), tree-sitter chunking, remote embedding, then `store::insert_batch` writes; keep that ordering so file deletions happen before inserting new chunks.
- Chunker heuristics in [cli/src/chunker.rs](../cli/src/chunker.rs) create an anchor chunk + semantic nodes capped by `MIN_CHUNK_LINES`/`MAX_CHUNK_LINES`; add languages via `get_language()` and ensure fallback chunking stays intact.
- Search path [cli/src/main.rs](../cli/src/main.rs) embeds the query once, calls `store::search`, and supports `--json` plus `--toon` Token-Oriented output; keep result structs backward compatible for MCP and hooks.

## Config, Storage & Runtime State
- Remote embeddings require config from [cli/src/config.rs](../cli/src/config.rs); `load()` caches the file/env via `OnceLock`, so always call `config::set_embedding_config` instead of writing files manually.
- Embedding client [cli/src/embeddings.rs](../cli/src/embeddings.rs) batches 10 texts with 100 ms sleeps and handles provider-specific response shapes; reuse `embed_batch_with_progress` to inherit retries/logging.
- SQLite schema + record struct live in [cli/src/store.rs](../cli/src/store.rs); the native addon mirrors it in [native/src/vector_store.rs](../native/src/vector_store.rs), so schema changes must touch both plus any FTS indexes.
- Data lives in `~/.osgrep/data/osgrep.db` (`get_db_path()`) and `.osgrep/server.json` is the lock for hot servers/watchers; tests override HOME to keep isolation.

## Integrations
- Watch mode (`watch_directory`) uses `notify` plus `is_code_file` extension filters; update that single list when enabling new filetypes.
- Claude plugin start hook [plugins/osgrep/hooks/start.js](../plugins/osgrep/hooks/start.js) launches `osgrep serve` on session start and logs to `/tmp/osgrep.log`; stop hook [plugins/osgrep/hooks/stop.js](../plugins/osgrep/hooks/stop.js) reads `.osgrep/server.json` to terminate cleanly.
- MCP server [mcp/src/main.rs](../mcp/src/main.rs) shells out to CLI commands; when CLI output changes, adjust `handle_*` responses so Json-RPC clients keep parsing results.

## Build & Test Workflows
- Standard Rust builds: `cargo build --release -p osgrep --features sqlite,parallel`, `cargo build -p osgrep-mcp`, and `cargo test` for unit coverage.
- Native addon builds via npm scripts in [native/package.json](../native/package.json) (`npm run build:sqlite`, `build:full`, `build:m2`, etc.); choose features that match the target agent.
- Vitest integration tests (e.g., [tests/integration/search.test.ts](../tests/integration/search.test.ts)) stub embeddings but hit the sqlite-backed store, so run them with `pnpm vitest --run` after rebuilding the addon.

## Conventions & Gotchas
- Always strip the repository root when storing paths (see `index_file()`), otherwise searches mix absolute and relative paths.
- Call `store::open()` before any CRUD so the sqlite-vec extension is registered once per process.
- Keep `#[cfg(feature = "...")]` gates aligned across crates; default CLI features include `sqlite` and `parallel`, while the N-API crate ships lean builds unless `build:full` is used.
- MCP/tools assume `osgrep config --init` has been run; surface actionable errors (as in `embeddings::init()`) instead of panics when the config is missing.
