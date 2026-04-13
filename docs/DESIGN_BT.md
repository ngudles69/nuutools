# Design: nuubt — Backtest Engine

## Overview

Fast historical replay engine for nuutrader. Greenfield — no existing Python code to replace.

**Binary:** `nuubt`
**Project location:** `/home/nuunuu/python/nuutools`
**Status:** Design deferred to a separate deep-dive session.

## Known Constraints

- Reads historical data from `workspace/data/backtest/`
- Writes results to `db/backtest.db`
- Reads strategy config from `settings.json5` backtest section
- Must be significantly faster than any Python alternative (the whole point)

## Deployment

Same pattern as nuucs — symlinked from nuutrader:

```
/home/nuunuu/python/nuutrader/nuutrader/
    nuubt → /home/nuunuu/python/nuutools/target/release/nuubt    # symlink
```

Config discovery via the same relative path mechanism (see [DESIGN_CS.md](DESIGN_CS.md)).

## Workspace Crate Structure

```
crates/
    common/         # Shared: config parsing, SQLite helpers, types
    backtest/       # nuubt binary: replay engine
```

## What This Is NOT

- Not a full Rust rewrite of nuutrader
- Not a trading execution engine (Python nuutrader handles that)
- Not a frontend (dashboard stays as-is)
- The Rust binary is a performance tool that slots into the existing Python ecosystem

---

*Split from DESIGN_BTCS.md — 2026-04-13*
