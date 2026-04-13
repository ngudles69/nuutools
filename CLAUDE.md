## Project

**NuuTools — Rust Companion for NuuTrader**

Performance-critical Rust tools that are drop-in replacements for their Python counterparts in nuutrader. Same interfaces, same data, same config — just faster.

**Two binaries:**
- `nuucs` — CollectScan pipeline (drop-in replacement for Python collectscan)
- `nuubt` — Backtest engine (greenfield, TBD)

**Design docs:** See `docs/DESIGN_CS.md` (CollectScan) and `docs/DESIGN_BT.md` (Backtest) for architecture, interfaces, and decisions.

### Constraints

- **Drop-in replacement**: nuucs must be interchangeable with Python collectscan. Same PID file, same API, same SQLite schema, same Redis messages, same config files
- **Config**: Reads nuutrader's `workspace/config/settings.json5` and `credentials.json5` at a known relative path from the binary
- **No business logic changes**: Same pipeline phases, same data flow, same signal format. Speed is the only difference
- **Deployment**: Binary symlinked from nuutrader into `target/release/`. `cargo build --release` updates production

### Reference Project

The Python collectscan at `/home/nuunuu/python/nuutrader/` is the authoritative reference. The Rust version must produce identical outputs for identical inputs.

**Before implementing any module:**
1. Read the Python source in `/home/nuunuu/python/nuutrader/backend/` for the relevant collector/scanner
2. Understand the data flow, SQLite schema, and Redis message format
3. Match the behavior exactly — then optimize for speed

## Technology Stack

| Technology | Version | Purpose |
|------------|---------|---------|
| tokio | 1.51 | Async runtime |
| sqlx | 0.8.6 | SQLite access (same schema as Python) |
| axum | 0.8.8 | HTTP API server (status, start, stop, list) |
| reqwest | 0.13.2 | HTTP client for CoinGlass, exchange APIs |
| serde | 1.0.228 | Serialization framework |
| json5 | 1.3.1 | Config parsing (settings.json5, credentials.json5) |
| redis | 1.0.4 | Signal publication to Redis channels (with `tokio-comp` feature for async) |
| rust_decimal | 1.41.0 | Financial numerics |
| tracing | 0.1.44 | Structured logging |
| tracing-subscriber | 0.3.23 | Log output formatting |
| chrono | 0.4.44 | Timestamps |
| clap | 4.6.0 | CLI argument parsing |
| thiserror | 2.0.18 | Typed errors in library crates |
| anyhow | 1.0.102 | Ergonomic errors in binary crates |
| tokio-tungstenite | 0.29.0 | WebSocket client (if needed for data feeds) |

## Conventions

Conventions not yet established. Will populate as patterns emerge during development.

## Architecture

See `docs/DESIGN_CS.md` and `docs/DESIGN_BT.md` for full architecture. Key points:
- Cargo workspace with 3 crates: `common`, `collectscan`, `backtest`
- Config discovery via known relative path from binary
- SQLite for data storage (same schema as Python)
- Redis for signal publication (same channels/format as Python)
- axum API for dashboard control (start/stop/status/list)

## Developer Profile

> Migrated from nuubot project.

| Dimension | Rating | Confidence |
|-----------|--------|------------|
| Communication | mixed | LOW |
| Decisions | deliberate-informed | MEDIUM |
| Explanations | concise | MEDIUM |
| Debugging | fix-first | MEDIUM |
| UX Philosophy | design-conscious | MEDIUM |
| Vendor Choices | opinionated | MEDIUM |
| Frustrations | scope-creep | HIGH |
| Learning | guided | MEDIUM |

**Directives:**
- **Communication:** Adapt response detail to match the complexity of each request. Brief for simple tasks, detailed for complex ones.
- **Decisions:** Present options in a structured comparison table with pros/cons. Let the developer make the final call.
- **Explanations:** Pair code with a brief explanation (1-2 sentences) of the approach. Keep prose minimal.
- **Debugging:** Prioritize the fix. Show the corrected code first, then optionally explain what was wrong. Minimize diagnostic preamble.
- **UX Philosophy:** Invest in UX quality: thoughtful spacing, smooth transitions, responsive layouts. Treat design as a first-class concern.
- **Vendor Choices:** Respect the developer's existing tool preferences. Ask before suggesting alternatives to their preferred stack.
- **Frustrations:** Do exactly what is asked -- nothing more. Follow instructions precisely. Keep explanations brief. Never break working code while fixing something else. All four frustration triggers are active.
- **Learning:** Explain concepts in context of the developer's codebase. Use their actual code as examples when teaching.
