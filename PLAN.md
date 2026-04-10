# Implementation Plan

Four phases (0-3), each independently testable. Each phase
builds on the previous one but produces a working, verifiable
feature on its own.

---

## Database Schema

The schema models the real-world hierarchy: a **concert** is
an event at a venue on a date. Multiple artists may perform at
the same concert. Each artist's appearance is a
**performance**, which contains **sets**, which contain
**songs**.

Hierarchy: `Concert -> Performances -> Sets -> Songs`
with `Artists` and `Venues` linked via Performances and
Concerts respectively.

### Design Decisions

- **Concert = date + venue.** One row per event. A festival
  date with 5 bands is one concert, not five.
- **Performance = artist at a concert.** This is where the
  setlist.fm URL and ID live, because setlist.fm models one
  setlist per artist per event.
- **Duplicate import detection** matches on
  `date + venue + artist`. Multiple artists at the same
  date/venue is expected (festival, multi-band bill).
- **Duplicate setlistfm_id handling:** update the existing
  performance row (and its sets/songs) rather than skip or
  error.

### Full CREATE TABLE Statements

```sql
-- ---------------------------------------------------------
-- Core tables
-- ---------------------------------------------------------

CREATE TABLE IF NOT EXISTS artists (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    mbid        TEXT UNIQUE,
    name        TEXT NOT NULL,
    sort_name   TEXT,
    url         TEXT,
    notes       TEXT DEFAULT '',
    created_at  TEXT DEFAULT (datetime('now')),
    updated_at  TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS venues (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    setlistfm_id    TEXT UNIQUE,
    name            TEXT NOT NULL,
    city            TEXT,
    state           TEXT,
    state_code      TEXT,
    country         TEXT,
    country_code    TEXT,
    latitude        REAL,
    longitude       REAL,
    url             TEXT,
    created_at      TEXT DEFAULT (datetime('now')),
    updated_at      TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS concerts (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    venue_id    INTEGER NOT NULL
                REFERENCES venues(id) ON DELETE CASCADE,
    event_date  TEXT NOT NULL,
    notes       TEXT DEFAULT '',
    created_at  TEXT DEFAULT (datetime('now')),
    updated_at  TEXT DEFAULT (datetime('now')),
    UNIQUE(venue_id, event_date)
);

CREATE TABLE IF NOT EXISTS performances (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    concert_id      INTEGER NOT NULL
                    REFERENCES concerts(id)
                    ON DELETE CASCADE,
    artist_id       INTEGER NOT NULL
                    REFERENCES artists(id)
                    ON DELETE CASCADE,
    setlistfm_id    TEXT UNIQUE,
    setlistfm_url   TEXT,
    tour_name       TEXT,
    info            TEXT,
    notes           TEXT DEFAULT '',
    created_at      TEXT DEFAULT (datetime('now')),
    updated_at      TEXT DEFAULT (datetime('now')),
    UNIQUE(concert_id, artist_id)
);

CREATE TABLE IF NOT EXISTS sets (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    performance_id  INTEGER NOT NULL
                    REFERENCES performances(id)
                    ON DELETE CASCADE,
    name            TEXT DEFAULT '',
    encore          INTEGER DEFAULT 0,
    position        INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS songs (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    set_id          INTEGER NOT NULL
                    REFERENCES sets(id)
                    ON DELETE CASCADE,
    name            TEXT NOT NULL,
    position        INTEGER NOT NULL,
    info            TEXT,
    is_tape         INTEGER DEFAULT 0,
    cover_artist_id INTEGER
                    REFERENCES artists(id),
    with_artist_id  INTEGER
                    REFERENCES artists(id),
    notes           TEXT DEFAULT ''
);

-- ---------------------------------------------------------
-- Full-text search (FTS5)
-- ---------------------------------------------------------

CREATE VIRTUAL TABLE IF NOT EXISTS search_index
    USING fts5(
    performance_id,
    artist_name,
    venue_name,
    city,
    song_names,
    tour_name,
    notes,
    content='',
    tokenize='unicode61'
);

-- ---------------------------------------------------------
-- Import tracking
-- ---------------------------------------------------------

CREATE TABLE IF NOT EXISTS import_jobs (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    input_date      TEXT NOT NULL,
    input_venue     TEXT,
    input_artist    TEXT,
    status          TEXT DEFAULT 'pending',
    error_message   TEXT,
    created_at      TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS import_job_matches (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    import_job_id   INTEGER NOT NULL
                    REFERENCES import_jobs(id)
                    ON DELETE CASCADE,
    setlistfm_id    TEXT NOT NULL,
    setlistfm_url   TEXT NOT NULL,
    artist_name     TEXT,
    artist_mbid     TEXT,
    venue_name      TEXT,
    event_date      TEXT,
    tour_name       TEXT,
    selected        INTEGER DEFAULT 1,
    performance_id  INTEGER
                    REFERENCES performances(id),
    created_at      TEXT DEFAULT (datetime('now'))
);

-- ---------------------------------------------------------
-- Indexes
-- ---------------------------------------------------------

CREATE INDEX IF NOT EXISTS idx_concerts_event_date
    ON concerts(event_date);
CREATE INDEX IF NOT EXISTS idx_concerts_venue_id
    ON concerts(venue_id);
CREATE INDEX IF NOT EXISTS idx_performances_concert_id
    ON performances(concert_id);
CREATE INDEX IF NOT EXISTS idx_performances_artist_id
    ON performances(artist_id);
CREATE INDEX IF NOT EXISTS idx_performances_setlistfm_id
    ON performances(setlistfm_id);
CREATE INDEX IF NOT EXISTS idx_sets_performance_id
    ON sets(performance_id);
CREATE INDEX IF NOT EXISTS idx_songs_set_id
    ON songs(set_id);
CREATE INDEX IF NOT EXISTS idx_songs_name
    ON songs(name);
CREATE INDEX IF NOT EXISTS idx_import_job_matches_job_id
    ON import_job_matches(import_job_id);
```

### Entity Relationship Diagram

```
+----------+     +-----------+
| artists  |     |  venues   |
+----------+     +-----------+
| id (PK)  |     | id (PK)   |
| mbid     |     | setlistfm |
| name     |     |   _id     |
| sort_name|     | name      |
| url      |     | city      |
| notes    |     | state     |
+----+-----+     | country   |
     |           +-----+-----+
     |                 |
     |   +-------------+-------------+
     |   |         concerts          |
     |   +---------------------------+
     |   | id (PK)                   |
     |   | venue_id (FK) ------->venues|
     |   | event_date               |
     |   | notes                    |
     |   | UNIQUE(venue_id,         |
     |   |        event_date)       |
     |   +------------+--------------+
     |                |
     |   +------------+--------------+
     +-->|      performances         |
         +---------------------------+
         | id (PK)                   |
         | concert_id (FK) ->concerts|
         | artist_id (FK) ->artists  |
         | setlistfm_id (unique)     |
         | setlistfm_url            |
         | tour_name                |
         | UNIQUE(concert_id,       |
         |        artist_id)        |
         +------------+--------------+
                      |
         +------------+--------------+
         |          sets             |
         +---------------------------+
         | id (PK)                   |
         | performance_id (FK)       |
         | name, encore, position    |
         +------------+--------------+
                      |
         +------------+--------------+
         |         songs             |
         +---------------------------+
         | id (PK)                   |
         | set_id (FK)               |
         | name, position            |
         | cover_artist_id (FK)      |
         | with_artist_id (FK)       |
         +---------------------------+
```

---

## Phase 0: Foundation

**Goal:** Project scaffolding, database, and basic helpers
so that Phase 1 has somewhere to land.

### Files to Create

```
setlist_collection_manager/
  app/
    __init__.py           # Flask app factory
    config.py             # Reads .env (API key, DB path)
    db.py                 # Connection mgmt, schema
                          #   creation, query helpers
  tests/
    __init__.py
    conftest.py           # Fixtures: in-memory DB,
                          #   Flask test client
    test_db.py            # Schema creation, helpers
  requirements.txt        # flask, httpx, python-dotenv
  requirements-dev.txt    # pytest, respx
  run.py                  # Entry point
  .env.example            # Template
```

### What This Phase Does

- Create the full schema (all tables) on first connection.
  All tables exist from day one, even if most stay empty
  until Phase 3. This avoids migrations between phases.
- Provide `db.py` helpers: `get_db()`, `query()`,
  `execute()`, `fetch_one()`.
- Provide `config.py` that loads `.env` and exposes
  settings as a dict or dataclass.

### Done When

- `pytest tests/test_db.py` passes
- `python run.py` starts Flask and returns 200 on `/`
- In-memory SQLite creates all tables without errors
- All foreign key constraints are verified (insert +
  cascade delete test)

---

## Phase 1: CSV Import

**Goal:** User provides a CSV of concerts and each row is
stored in `import_jobs` for later processing.

### Interface

CLI first:

```bash
python -m app.csv_import path/to/concerts.csv
```

Outputs a summary to stdout:

```
Imported: 12
Skipped (duplicate): 2
Errors: 1
  Row 7: invalid date "July 4th"
```

### CSV Format

```
date,venue[,artist]
2023-07-01,Madison Square Garden
2023-03-15,The Fillmore,Phoebe Bridgers
2022-11-20,Red Rocks Amphitheatre
2023-07-01,Madison Square Garden,Foo Fighters
```

- `date`: `YYYY-MM-DD` (ISO 8601)
- `venue`: free-text venue name, required
- `artist`: optional, aids disambiguation; multiple
  rows with the same date+venue but different artists
  are expected and normal

### Database Tables Involved

Only `import_jobs`. No other tables written in Phase 1.

### Files to Create

| File | Purpose |
|------|---------|
| `app/csv_import.py` | Parse CSV, validate, insert |
| `app/__main__.py` | CLI entry: `python -m app` |
| `tests/test_csv_import.py` | Unit tests |
| `tests/fixtures/sample.csv` | Test CSV file |

### Key Functions

```python
def parse_csv(file_path: str) -> list[dict]:
    """
    Read CSV, validate each row. Returns list of
    {date, venue, artist} dicts.
    Raises on unrecoverable format errors.
    """

def import_concerts(db, rows: list[dict]) -> dict:
    """
    Insert rows into import_jobs. Returns summary:
    {imported: N, skipped: N, errors: [...]}.
    Duplicate = same date + venue + artist already
    exists in import_jobs (any status).
    """
```

### Validation Rules

- Date must parse as `YYYY-MM-DD`
- Venue must be non-empty after stripping whitespace
- Artist is optional; empty string stored as NULL
- No normalization yet -- store exactly as typed

### Done When

- `pytest tests/test_csv_import.py` passes:
  - Valid 3-row CSV imports correctly
  - Missing date rejected
  - Missing venue rejected
  - Duplicate `date+venue+artist` rows skipped
  - Different artists at same date+venue both import
  - Optional artist column works (2-col and 3-col)
  - Empty file produces zero imports
- `python -m app.csv_import tests/fixtures/sample.csv`
  inserts rows with status `pending`

---

## Phase 2: setlist.fm URL Retrieval

**Goal:** For each `pending` import job, search the
setlist.fm API by date and venue. Store every matching
setlist URL. Do NOT fetch full song data yet.

### How It Works

1. Read all `import_jobs` with status `pending`.
2. For each job, search setlist.fm by date + venue (and
   artist if provided).
3. A date+venue search typically returns multiple results
   -- one per artist who played that event. **Auto-select
   all of them.** Each result is a different artist's
   performance at the same concert.
4. Store each result as a row in `import_job_matches`
   with `selected = 1`.
5. Update the import job status to `matched` or
   `not_found`.

### Database Tables Involved

- **`import_jobs`** -- status updated to `matched`,
  `not_found`, or `error`
- **`import_job_matches`** -- one row per API result per
  job, all auto-selected

The `import_job_matches` table stores enough data from
the API response to identify each match without a second
API call:

- `setlistfm_id` / `setlistfm_url` -- for Phase 3 fetch
- `artist_name` / `artist_mbid` -- for display and
  deduplication
- `venue_name` / `event_date` -- for verification
- `tour_name` -- useful context
- `selected` -- defaults to 1 (auto-select all)
- `performance_id` -- NULL until Phase 3 populates it

### Files to Create

| File | Purpose |
|------|---------|
| `app/setlistfm_client.py` | API client class |
| `app/url_retriever.py` | Orchestration logic |
| `tests/test_setlistfm_client.py` | Mocked HTTP tests |
| `tests/test_url_retriever.py` | Integration tests |
| `tests/fixtures/api_responses/` | JSON fixtures |

### Key Functions

```python
# url_retriever.py

def retrieve_urls(db, client, job_ids=None) -> dict:
    """
    For each pending import_job (or specific IDs),
    search setlist.fm. Store matches. Update status.
    Returns {matched: N, not_found: N, errors: N}.
    """

def _process_one_job(db, client, job) -> str:
    """
    Search API for one job. Insert all results into
    import_job_matches with selected=1.
    Return new status string.
    """
```

### Search Strategy

For each import job:

1. Convert date to `dd-MM-yyyy`.
2. Search with `date` + `venueName`. If `input_artist`
   is set, include `artistName` too.
3. If zero results with venue name, retry with date
   only -- but only if `input_artist` was provided (use
   `artistName` alone). Otherwise mark `not_found`.
4. Store ALL results as matches. Do not filter or prompt.

### Idempotency

- Only process jobs with status `pending`.
- Running again skips already-processed jobs.
- To reprocess a job, reset its status to `pending` and
  delete its `import_job_matches` rows.

### Done When

- `pytest tests/test_setlistfm_client.py` passes:
  - Successful search returns parsed results
  - Rate limiting sleeps between requests
  - 429 triggers retry with exponential backoff
  - Daily limit raises exception
  - Empty results handled
- `pytest tests/test_url_retriever.py` passes:
  - Single match creates one `import_job_matches` row
  - Multiple matches (festival) creates N rows, all
    with `selected = 1`
  - Zero matches sets status to `not_found`
  - API error sets status to `error` with message
  - Already-matched jobs skipped
  - Date format conversion correct
- Manual test against real API shows URLs in
  `import_job_matches`

---

## Phase 3: Full Data Population

**Goal:** For each selected match in `import_job_matches`,
fetch the full setlist and populate `artists`, `venues`,
`concerts`, `performances`, `sets`, and `songs`.

### How It Works

1. Read all `import_job_matches` where `selected = 1`
   and `performance_id IS NULL`.
2. Group matches by `event_date + venue_name` to
   identify which ones belong to the same concert.
3. For each group:
   a. Upsert the venue into `venues`.
   b. Upsert or find the concert in `concerts`
      (by `venue_id + event_date`).
   c. For each match in the group:
      - Fetch full setlist via `GET /setlist/{id}`.
      - Upsert the artist into `artists`.
      - Upsert the performance into `performances`.
        If `setlistfm_id` already exists, **update**
        the existing row and delete its old sets/songs.
      - Insert sets and songs.
      - Set `import_job_matches.performance_id`.
4. Update `import_jobs.status` to `populated`.

### Upsert Logic

- **Artists:** upsert by `mbid`. If no `mbid` (rare),
  match by exact `name`.
- **Venues:** upsert by `setlistfm_id`.
- **Concerts:** upsert by `venue_id + event_date`
  (the UNIQUE constraint).
- **Performances:** if `setlistfm_id` exists, delete
  old sets/songs (CASCADE), update the row. Otherwise
  insert.

### Files to Create

| File | Purpose |
|------|---------|
| `app/data_populator.py` | Fetch + upsert logic |
| `tests/test_data_populator.py` | Mocked API tests |
| `tests/fixtures/api_responses/setlist_full.json` | Fixture |

### Key Functions

```python
# data_populator.py

def populate_matches(db, client) -> dict:
    """
    Process all selected, unpopulated matches.
    Returns {populated: N, errors: N}.
    """

def _populate_group(db, client, matches) -> int:
    """
    Populate one concert's worth of matches.
    Returns concert_id.
    """

def _upsert_artist(db, artist_data) -> int:
    """Upsert by mbid, return artist.id."""

def _upsert_venue(db, venue_data) -> int:
    """Upsert by setlistfm_id, return venue.id."""

def _upsert_concert(db, venue_id, event_date) -> int:
    """Upsert by venue_id+event_date, return id."""

def _upsert_performance(
    db, concert_id, artist_id, setlist_data
) -> int:
    """
    Upsert by setlistfm_id. If exists, delete old
    sets/songs and update. Return performance.id.
    """

def _insert_sets_and_songs(
    db, performance_id, sets_data
) -> None:
    """Insert sets and songs for a performance."""
```

### Transaction Strategy

Each concert group (concert + all its performances +
sets + songs) is wrapped in a single transaction. If
any part fails, the whole group rolls back.

### Edge Cases

- **Empty setlists.** Create the performance row but
  skip sets/songs. The UI shows "No setlist available."
- **Cover songs.** The `cover` field references another
  artist. Upsert that artist and set `cover_artist_id`.
- **Guest artists.** The `with` field references another
  artist. Upsert and set `with_artist_id`.
- **Same artist at multiple concerts.** Normal -- the
  artist row is reused, a new performance row is created.

### Done When

- `pytest tests/test_data_populator.py` passes:
  - Full setlist populates all tables correctly
  - Multiple performances at same concert share one
    `concerts` row
  - Artist upsert: existing reused, new created
  - Venue upsert: existing reused, new created
  - Performance with existing `setlistfm_id` updates
    (old sets/songs deleted, new ones inserted)
  - Cover song creates cover artist link
  - Guest artist creates with-artist link
  - Empty setlist creates performance, no sets/songs
  - `import_job_matches.performance_id` set
  - Transaction rollback on error leaves DB clean
- After running all phases on real data:
  - `concerts` rows have valid venue FKs
  - `performances` rows link concerts to artists
  - `sets` and `songs` have correct ordering
  - Multiple artists at the same date/venue share one
    concert row

---

## Execution Order

```
Phase 0 --> Phase 1 --> Phase 2 --> Phase 3
scaffold    CSV import   find URLs    populate
& DB        to import    via API      full data
            _jobs
```

Each phase produces testable output:
- Phase 0: empty app runs, DB schema correct
- Phase 1: CSV rows in `import_jobs`
- Phase 2: setlist.fm URLs in `import_job_matches`
- Phase 3: full data in core tables

No phase depends on the web UI. The UI can be built
after all phases are complete.
