# HoopKicks

HoopKicks is a data engineering capstone project that ingests NBA shoe-wear data (per player, per game),
persists it in an analytical store, and publishes weekly recap tables + insights.

Design goals:

- DRY + SOLID architecture (easy to add WNBA / EuroLeague later)
- Idempotent runs (safe re-runs and backfills)
- Observability (logs, manifests, counts)
- “Ship-ready” evolution path (Docker, scheduling, eventually AWS)

## Data sources

Primary:

- Kixstats (scraped)

Optional enrichment:

- NBA advanced stats provider (pluggable interface; start small)

## Scraping etiquette + reliability (non-negotiable)

This project defaults to **slow, cached, incremental** scraping from day one:

- Cache every fetched response (URL + date keyed)
- Backoff and retry on transient failures
- Respect 429 responses (and `Retry-After` when present)
- Backfills run in small batches with resume support

Reason: during early exploration, Kixstats endpoints returned **HTTP 429 Too Many Requests** when hit too aggressively.

## Architecture

### High-level flow

1) Discover games for a date range
2) Fetch + cache pages
3) Parse → raw entities
4) Transform → canonical domain models
5) Load → warehouse
6) Publish → weekly recap tables + optional API

### Package layout

```plaintext
HoopKicks
├─ LICENSE
├─ README.md
├─ data
│  ├─ fixtures
│  └─ samples
├─ docs
│  └─ adr
├─ pyproject.toml
├─ src
│  └─ hoopkicks
│     ├─ __init__.py
│     ├─ analytics
│     │  └─ rollups.py
│     ├─ cli
│     │  └─ main.py
│     ├─ config
│     ├─ domain
│     ├─ pipelines
│     │  ├─ enrich.py
│     │  ├─ ingest.py
│     │  └─ publish.py
│     ├─ sources
│     │  ├─ kixstats
│     │  └─ nba_stats
│     └─ storage
│        ├─ duckdb.py
│        ├─ files.py
│        └─ postgres.py
└─ tests
```

## Data model (canonical)

Dimensions:

- league, season, team, player, brand, shoe_model, shoe_colourway

Facts:

- player_game_shoe (minutes, points, rebounds, assists, steals, blocks, shoe worn)
- player_game_advanced (optional enrichment)

## Outputs

- Daily: top shoes worn, top performers by shoe
- Weekly: recap tables (league-wide and per player)
- Optional: read-only API endpoints (later)

## CLI usage

Examples (final flags may change as the CLI matures):

- Ingest a day:
  - `hoopkix ingest --league nba --date 2025-12-16`

- Backfill a date range:
  - `hoopkix ingest --league nba --start 2025-10-01 --end 2025-10-31`

- Publish weekly recap:
  - `hoopkix publish --league nba --week 2025-W42`

## Milestones (tracked in docs)

- [ ] MVP: scrape games + store raw JSON + basic rollups
- [ ] Warehouse: SQLite/DuckDB curated tables
- [ ] Enrichment: plug in advanced stats provider
- [ ] Reporting: weekly recap output (CSV + charts)
- [ ] DevOps (final compulsory): Docker + CI + scheduling + deployment notes

## Licence

Apache 2.0
