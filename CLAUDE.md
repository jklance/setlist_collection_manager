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

**Status:** Pre-implementation. `DESIGN.md` contains the full
specification including database schema, API integration details,
directory structure, and phased implementation plan.

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

# Configuration
cp .env.example .env  # then add your setlist.fm API key
```

## Architecture

**Data flow:** Browser -> Flask routes -> db.py / importer.py /
search.py -> SQLite (+ setlistfm_client.py -> setlist.fm API)

**Key modules** (under `app/`):
- `db.py` — Connection management, schema creation,
  `query()`/`execute()`/`fetch_one()` helpers
- `setlistfm_client.py` — All setlist.fm API HTTP calls, rate
  limiting (1 req/s, 1440/day), retry on 429
- `importer.py` — Orchestrates import workflow: parse user input
  -> API search -> review matches -> upsert data -> update FTS
- `search.py` — FTS5 search helpers
- `routes/` — Flask blueprints: concerts (browse/view/edit),
  import_routes, search_routes, api (internal JSON)

**Database:** 6 core tables: `artists`, `venues`, `concerts`,
`sets`, `songs`, `import_jobs`, plus `search_index` (FTS5 virtual
table). All FKs use `ON DELETE CASCADE`. Full schema is in
`DESIGN.md` section 4.

**Import workflow** is a multi-step process: user submits
date/venue pairs -> API search -> review/confirm matches -> upsert
artist/venue/concert/sets/songs -> update FTS index -> update
import_job status.

## Domain Terminology

- **Performance** = one artist's set at one venue on one date
- **Show** = all performances at one venue on one date
- **Concert** (DB table) = a single performance (one artist, one
  venue, one date) linked to a setlist.fm entry
- **Set** = a group of songs within a concert (e.g., "Set 1",
  "Encore")

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
