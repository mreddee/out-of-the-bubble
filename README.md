# Out of the Bubble

> Step outside your listening bubble — on purpose, with a map.

**Status:** 🚧 early development (Phase 0 — setup).

## The problem
Recommendation algorithms favour popular artists and rarely explain *why* they suggest something.
Listeners who really want to explore have no knobs and no transparency.

## What it does
- **Open to anyone in 30 seconds** — no account, three precise questions.
- **Obscurity dial** — choose how little-known the artists should be.
- **Explained bridge** — every suggestion shows the path from your artists to the new one.
- **Measures whether discovery lasts** — with a linked Last.fm / ListenBrainz history,
  we check after 4 weeks whether the artist actually entered your listening.

## Architecture
ListenBrainz / MusicBrainz → BigQuery (exploration) → Databricks (bronze/silver/gold, ALS model)
→ dbt (tested final tables) → Parquet on Hugging Face → Streamlit app with DuckDB.

## Development
```bash
uv sync                 # create .venv with Python 3.11+ and dev tools
uv run pytest           # tests
uv run ruff check .     # lint
uv run ruff format .    # format
cp .env.example .env    # then fill in your keys
```

## Data & licences
Listening data from [ListenBrainz](https://listenbrainz.org) (CC0) and artist metadata from
[MusicBrainz](https://musicbrainz.org). Non-commercial project.
