# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when
working with code in this repository.

## Project Overview

Setlist Collection Manager is a personal tool for collecting and
browsing concert setlists. Users provide concert dates and venues,
the app queries the setlist.fm API, stores data in SQLite, and
serves a Flask web UI. The full design is in `DESIGN.md`.

**Stack:** Python 3.10+ / Flask / SQLite / vanilla HTML+CSS+JS
(Pico CSS) / Jinja2 templates. No ORM — raw SQL with parameterized
queries via a thin `db.py` helper module.

**Status:** Pre-implementation. `DESIGN.md` contains the original
specification. `PLAN.md` contains the revised database schema and
phased implementation plan (Phases 0-3).

## Commands

```bash
# Run the app
python run.py              # starts Flask on localhost:5000

# Install dependencies
pip install -r requirements.txt      # flask, httpx, python-dotenv
pip install -r requirements-dev.txt  # pytest, respx

# Run tests
pytest                           # all tests
pytest tests/test_db.py          # single test file
pytest tests/test_db.py::test_name -v  # single test

# Import concerts from CSV (Phase 1)
python -m app.csv_import path/to/concerts.csv

# Configuration
cp .env.example .env  # then add your setlist.fm API key
```

## Architecture

**Data flow:** Browser -> Flask routes -> db.py / importer.py /
search.py -> SQLite (+ setlistfm_client.py -> setlist.fm API)

**Key modules** (under `app/`):
- `db.py` -- Connection management, schema creation,
  `query()`/`execute()`/`fetch_one()` helpers
- `config.py` -- Reads `.env`, exposes settings
- `csv_import.py` -- Phase 1: parse CSV, insert import_jobs
- `setlistfm_client.py` -- All setlist.fm API HTTP calls,
  rate limiting, retry on 429
- `url_retriever.py` -- Phase 2: search API, store matches
- `data_populator.py` -- Phase 3: fetch full setlists,
  upsert all entities
- `search.py` -- FTS5 search helpers
- `routes/` -- Flask blueprints (built after Phases 0-3)

**Database:** 8 tables: `artists`, `venues`, `concerts`,
`performances`, `sets`, `songs`, `import_jobs`,
`import_job_matches`, plus `search_index` (FTS5 virtual
table). All FKs use `ON DELETE CASCADE`. Revised schema is
in `PLAN.md`.

**Import workflow** is a 3-phase CLI pipeline:
Phase 1: CSV -> `import_jobs` rows.
Phase 2: API search -> `import_job_matches` rows.
Phase 3: fetch full data -> populate core tables.

## Domain Terminology

- **Concert** (DB table) = one event: a date + venue pair.
  A festival date with 5 bands is one concert row.
- **Performance** (DB table) = one artist's appearance at a
  concert. Holds the setlist.fm URL/ID. One row per artist
  per concert.
- **Set** = a group of songs within a performance (e.g.,
  "Set 1", "Encore")
- **Song** = one track within a set, with position, optional
  cover/guest artist links

Hierarchy: Concert -> Performances -> Sets -> Songs

## Important Conventions

- Markdown docs should wrap at 80 characters max line length
- setlist.fm API dates use `dd-MM-yyyy` format; store internally
  as `YYYY-MM-DD` (ISO 8601)
- All SQL must use parameterized queries (`?` placeholders) —
  never string interpolation
- setlist.fm attribution links (`setlistfm_url`) must always be
  displayed in the UI
- API key lives in `.env` (gitignored); loaded via `python-dotenv`
- Flask dev server binds to `127.0.0.1` only
