# Design: nuucs — CollectScan

## Overview

Drop-in Rust replacement for Python `collectscan`. Collects market data, computes indicators, runs scanners, publishes signals. Same interfaces, same data, same config — just faster.

**Binary:** `nuucs`
**Project location:** `/home/nuunuu/python/nuutools`

## Core Principle

**Drop-in replacement.** You can run the Python version or the Rust version interchangeably. Same PID file, same API, same SQLite schema, same Redis messages, same config files. The only difference is speed.

## Architecture

### Deployment Layout

```
/home/nuunuu/python/nuutrader/nuutrader/
    nuucs → /home/nuunuu/python/nuutools/target/release/nuucs    # symlink
    workspace/
        config/
            settings.json5          # THE config — both Python and Rust read this
            credentials.json5       # API keys (CoinGlass, exchanges)
            telegram.json5          # Notification channels
        data/
            ohlcv/                  # Per-symbol SQLite DBs
            vp/                     # Volume profiles
            oi/                     # Open interest
            indicators/             # Computed indicators
        db/
            signals.db              # Signal history
            collectscan.db          # Run tracking
            coinglass.db            # Liquidation/funding
            meta.db                 # Asset metadata
        logs/
            collectscan/            # Per-run log directories
```

### Config Discovery

Binary discovers config via **known relative path**:
- Binary location: `nuutrader/nuutrader/nuucs`
- Config location: `nuutrader/nuutrader/workspace/config/settings.json5`
- Relative path: `workspace/config/settings.json5` from binary's directory

From `settings.json5`, all other paths (data dirs, DB locations, Redis URL, log paths) are discovered. No env vars or CLI flags needed for standard deployment.

### Dev Layout

```
/home/nuunuu/python/nuutools/           # Rust project (separate Claude Code session)
    Cargo.toml                          # Workspace root
    settings.json5 → symlink            # Points to nuutrader's real config
    crates/
        common/                         # Shared types, config parsing, SQLite access
        collectscan/                    # nuucs binary
```

Symlink mirrors the production relative path so the binary works identically in dev and production.

## Invocation Modes

- `nuucs` — Daemon mode. Starts up, serves API, runs pipelines on schedule or on-demand
- `nuucs run [SYM ...]` — One-shot mode. Runs one pipeline cycle and exits (manual/debug use)
- `nuucs setup` — Create/verify system cron entries from settings.json5

## API (same as Python)

HTTP API served during execution. Same port as Python version (from settings/env).

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/status` | GET | Pipeline progress (phase, symbol, elapsed, progress %) |
| `/start` | POST | Trigger a pipeline cycle (optional: specific symbols) |
| `/stop` | POST | Graceful shutdown (finish current symbol, then stop) |
| `/list` | GET | Last N runs from collectscan.db |

## CLI Integration

The Python `nuubot` CLI communicates with nuucs via API:

| CLI Command | What Happens |
|-------------|-------------|
| `nuubot collectscan start [SYM]` | Check if nuucs running (ping API). If not, spawn process + store PID. Then POST `/start` |
| `nuubot collectscan stop` | POST `/stop` to API |
| `nuubot collectscan kill` | SIGKILL to PID from file + clear lock |
| `nuubot collectscan status` | GET `/status` from API |
| `nuubot collectscan list [N]` | GET `/list?n=N` from API |
| `nuubot collectscan setup` | Calls `nuucs setup` to configure system cron |
| `nuubot collectscan clear` | Clear stale PID/lock file |

Aliases: `cs`, `c` (e.g., `nuubot cs start`)

## Process Management

- **PID file:** Same location and name as Python version (`.collectscan.pid`)
- **Singleton:** flock-based, same pattern as Python
- **Cron:** Reads schedule from settings.json5 (e.g., `1,31 * * * *`), updates system crontab via `nuucs setup`

## Pipeline Phases

Same pipeline as Python:
1. CoinGlass BTC data
2. CoinGlass Macro data (liquidations, funding, OI)
3. BTC Regime detection (1d EMA34 bull/bear)
4. OI Cache update (multi-exchange)
5. Per-symbol collection: OHLCV → indicators (EMA34, volume EMA, ATR) → Volume Profile → regime scan → VP scan → signals

## Data Interfaces (Drop-In Compatible)

**SQLite — same schema, same files:**
- `data/ohlcv/{symbol}.db` — per-symbol OHLCV bars
- `data/vp/{symbol}.db` — volume profile snapshots
- `data/oi/{symbol}.db` — open interest data
- `data/indicators/{symbol}.db` — computed indicators
- `db/signals.db` — signal history with cooldown enforcement
- `db/collectscan.db` — run tracking (id, start_time, end_time, status, mode, error, row_count, pid, last_symbol, run_dir)
- `db/coinglass.db` — liquidation/funding data
- `db/meta.db` — asset metadata tiers

**Redis — same channels, same message format:**
- Publishes signals on the same channels Python uses
- Same JSON message structure
- Python botmanager reads them without changes

## Technology Stack

| Technology | Purpose |
|------------|---------|
| tokio | Async runtime |
| sqlx (SQLite) | Database access — same schema as Python |
| axum | HTTP API server (status, start, stop, list) |
| reqwest | HTTP client for CoinGlass, exchange APIs |
| serde + json5 | Config parsing (settings.json5, credentials.json5) |
| redis (crate) | Signal publication |
| rust_decimal | Financial numerics |
| tracing | Structured logging |
| chrono | Timestamps |
| clap | CLI argument parsing |

## Workspace Crate Structure

```
crates/
    common/         # Shared: config parsing, SQLite helpers, types, Redis client
    collectscan/    # nuucs binary: pipeline, API server, scheduling
```

---

*Split from DESIGN_BTCS.md — 2026-04-13*
