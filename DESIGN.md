# Setlist Collection Manager -- System Design

## 1. Executive Summary

A personal tool for collecting and browsing concert setlists. The user provides
concert dates and venues, the application queries the setlist.fm API to retrieve
setlist data, stores everything locally in SQLite, and serves a web UI for
browsing, searching, and editing. The entire stack is Python + Flask + SQLite +
vanilla HTML/CSS/JS -- no build tools, no containers, no external services beyond
the setlist.fm API itself.

---

## 2. Requirements Analysis

### Functional Requirements

| # | Requirement |
|---|-------------|
| F1 | Accept a list of concert dates and venue names as input |
| F2 | Search the setlist.fm API to find matching setlists |
| F3 | Retrieve and store: setlist.fm page URL, artists, songs (in order), venue, date |
| F4 | Browse all collected setlists |
| F5 | Search setlists by artist, song, venue, date |
| F6 | Edit stored data (correct errors, add personal notes) |

### Non-Functional Requirements

| # | Requirement |
|---|-------------|
| N1 | Single-user, runs locally |
| N2 | Minimal dependencies -- easy to install and run |
| N3 | Data persists in a single portable file (SQLite) |
| N4 | Respects setlist.fm API rate limits |
| N5 | Works offline once data is fetched |

### Constraints

- Personal tool: one user, no auth needed on the web UI
- setlist.fm API key required (free for non-commercial use)
- API rate limit: 1 request/second, 1,440 calls/day

---

## 3. Technology Stack

### Recommendation

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Language** | Python 3.10+ | Widely known, excellent HTTP/DB libraries, fast to develop |
| **Web Framework** | Flask | Minimal, no magic, perfect for single-user tools |
| **Database** | SQLite 3 (via `sqlite3` stdlib) | Zero setup, single file, portable, powerful enough for millions of rows |
| **ORM** | None -- raw SQL with a thin helper module | Keeps it simple; SQLite SQL is straightforward and the schema is small |
| **HTTP Client** | `httpx` | Modern, supports rate limiting middleware, sync API |
| **Templating** | Jinja2 (built into Flask) | Server-rendered HTML, no JS framework needed |
| **Frontend** | Vanilla HTML + CSS + minimal JS | No build step. Use a classless CSS framework like Pico CSS for aesthetics |
| **Search** | SQLite FTS5 | Full-text search built into SQLite, no external engine |
| **Config** | `.env` file loaded via `python-dotenv` | Simple, keeps the API key out of code |

### Alternatives Considered

| Alternative | Why Not |
|-------------|---------|
| Django | Too heavy for a personal tool with 6 tables |
| PostgreSQL | Requires a running server; SQLite is file-based and sufficient |
| React/Vue SPA | Adds a build toolchain; server-rendered HTML is simpler for this scope |
| SQLAlchemy | Useful for large projects but adds abstraction overhead for a small schema |
| Electron | A browser tab is enough; no need for a desktop wrapper |

---

## 4. Database Schema

The schema mirrors the setlist.fm data model but is flattened for simplicity.
All foreign keys use `ON DELETE CASCADE` so deleting a concert removes its songs.

```sql
-- -----------------------------------------------------------
-- Core tables
-- -----------------------------------------------------------

CREATE TABLE IF NOT EXISTS artists (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    mbid        TEXT UNIQUE,                    -- MusicBrainz ID from setlist.fm
    name        TEXT NOT NULL,
    sort_name   TEXT,
    url         TEXT,                           -- setlist.fm artist page
    created_at  TEXT DEFAULT (datetime('now')),
    updated_at  TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS venues (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    setlistfm_id    TEXT UNIQUE,                -- setlist.fm venue ID
    name            TEXT NOT NULL,
    city            TEXT,
    state           TEXT,
    state_code      TEXT,
    country         TEXT,
    country_code    TEXT,
    latitude        REAL,
    longitude       REAL,
    url             TEXT,                        -- setlist.fm venue page
    created_at      TEXT DEFAULT (datetime('now')),
    updated_at      TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS concerts (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    setlistfm_id    TEXT UNIQUE,                -- setlist.fm setlist ID
    artist_id       INTEGER NOT NULL REFERENCES artists(id) ON DELETE CASCADE,
    venue_id        INTEGER NOT NULL REFERENCES venues(id) ON DELETE CASCADE,
    event_date      TEXT NOT NULL,              -- ISO 8601: YYYY-MM-DD
    tour_name       TEXT,
    setlistfm_url   TEXT,                       -- link to setlist.fm page (required attribution)
    info            TEXT,                        -- setlist.fm's info field
    notes           TEXT DEFAULT '',            -- user's personal notes
    created_at      TEXT DEFAULT (datetime('now')),
    updated_at      TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS sets (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    concert_id  INTEGER NOT NULL REFERENCES concerts(id) ON DELETE CASCADE,
    name        TEXT DEFAULT '',                -- e.g. "Set 1", "Encore"
    encore      INTEGER DEFAULT 0,             -- encore number (0 = main set)
    position    INTEGER NOT NULL DEFAULT 0     -- ordering within the concert
);

CREATE TABLE IF NOT EXISTS songs (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    set_id      INTEGER NOT NULL REFERENCES sets(id) ON DELETE CASCADE,
    name        TEXT NOT NULL,
    position    INTEGER NOT NULL,              -- order within the set
    info        TEXT,                          -- e.g. "acoustic version"
    is_tape     INTEGER DEFAULT 0,            -- played from tape, not live
    cover_artist_id INTEGER REFERENCES artists(id),  -- if this is a cover
    with_artist_id  INTEGER REFERENCES artists(id),  -- guest artist
    notes       TEXT DEFAULT ''               -- user's personal notes
);

-- -----------------------------------------------------------
-- Full-text search (FTS5) virtual table
-- -----------------------------------------------------------

CREATE VIRTUAL TABLE IF NOT EXISTS search_index USING fts5(
    concert_id,
    artist_name,
    venue_name,
    city,
    song_names,
    tour_name,
    notes,
    content='',                               -- contentless: we manage sync
    tokenize='unicode61'
);

-- -----------------------------------------------------------
-- Import tracking (to handle bulk imports and retries)
-- -----------------------------------------------------------

CREATE TABLE IF NOT EXISTS import_jobs (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    input_date      TEXT NOT NULL,             -- the date the user provided
    input_venue     TEXT,                      -- the venue name the user provided
    input_artist    TEXT,                      -- optional: artist name if provided
    status          TEXT DEFAULT 'pending',    -- pending | matched | not_found | error
    matched_concert_id INTEGER REFERENCES concerts(id),
    error_message   TEXT,
    created_at      TEXT DEFAULT (datetime('now'))
);

-- -----------------------------------------------------------
-- Indexes
-- -----------------------------------------------------------

CREATE INDEX IF NOT EXISTS idx_concerts_event_date ON concerts(event_date);
CREATE INDEX IF NOT EXISTS idx_concerts_artist_id ON concerts(artist_id);
CREATE INDEX IF NOT EXISTS idx_concerts_venue_id ON concerts(venue_id);
CREATE INDEX IF NOT EXISTS idx_songs_set_id ON songs(set_id);
CREATE INDEX IF NOT EXISTS idx_songs_name ON songs(name);
CREATE INDEX IF NOT EXISTS idx_sets_concert_id ON sets(concert_id);
```

### Entity Relationship Diagram (ASCII)

```
  +-----------+       +-----------+       +----------+
  |  artists  |       |  venues   |       |  songs   |
  +-----------+       +-----------+       +----------+
  | id (PK)   |       | id (PK)   |       | id (PK)  |
  | mbid       |<--+   | setlistfm_id    | name     |
  | name       |   |   | name      |       | position |
  | sort_name  |   |   | city      |       | info     |
  | url        |   |   | state     |       | is_tape  |
  +-----+------+   |   | country   |       +----+-----+
        |           |   +-----+-----+            |
        |           |         |                  |
        |   +-------+---------+------+     +-----+-----+
        +-->|       concerts         |     |   sets    |
            +------------------------+     +-----------+
            | id (PK)               |<----| concert_id|
            | setlistfm_id          |     | name      |
            | artist_id (FK)        |     | encore    |
            | venue_id (FK)         |     | position  |
            | event_date            |     +-----+-----+
            | tour_name             |           |
            | setlistfm_url         |           |
            | notes (user editable) |     songs.set_id (FK)
            +------------------------+
                      |
              +-------+--------+
              |  import_jobs   |
              +----------------+
              | input_date     |
              | input_venue    |
              | status         |
              +----------------+
```

---

## 5. Application Architecture

### Directory Structure

```
setlist_collection_manager/
|-- app/
|   |-- __init__.py              # Flask app factory
|   |-- config.py                # Configuration (reads .env)
|   |-- db.py                    # Database connection, migrations, helpers
|   |-- models.py                # Data classes / named tuples for rows
|   |-- setlistfm_client.py     # setlist.fm API client (all HTTP calls)
|   |-- importer.py             # Import orchestration logic
|   |-- search.py               # FTS5 search helpers
|   |-- routes/
|   |   |-- __init__.py
|   |   |-- concerts.py          # Browse, view, edit concert/setlist pages
|   |   |-- import_routes.py     # Import workflow pages
|   |   |-- search_routes.py     # Search page
|   |   |-- api.py               # Internal JSON API (for JS interactions)
|   |-- templates/
|   |   |-- base.html            # Layout with nav
|   |   |-- concerts/
|   |   |   |-- index.html       # Concert list (browse)
|   |   |   |-- detail.html      # Single concert + setlist
|   |   |   |-- edit.html        # Edit concert / songs
|   |   |-- import/
|   |   |   |-- form.html        # Input form (dates + venues)
|   |   |   |-- review.html      # Review API matches before saving
|   |   |   |-- status.html      # Import progress / results
|   |   |-- search/
|   |   |   |-- results.html     # Search results
|   |   |-- partials/            # HTMX-friendly partial templates
|   |       |-- song_row.html
|   |       |-- concert_card.html
|   |-- static/
|       |-- style.css            # Custom overrides on top of Pico CSS
|       |-- app.js               # Minimal JS (inline editing, search-as-you-type)
|-- tests/
|   |-- conftest.py
|   |-- test_setlistfm_client.py
|   |-- test_importer.py
|   |-- test_db.py
|   |-- test_routes.py
|-- .env.example                 # Template for API key config
|-- .gitignore
|-- requirements.txt
|-- run.py                       # Entry point: python run.py
|-- README.md
```

### Component Interaction Diagram

```
  User (Browser)
       |
       | HTTP (localhost:5000)
       v
  +----+----+
  |  Flask  |  (routes/ -- handles requests, renders templates)
  +----+----+
       |
       +------------+--------------+
       |            |              |
       v            v              v
  +--------+  +-----------+  +---------+
  | db.py  |  | importer  |  | search  |
  +--------+  +-----------+  +---------+
  | SQLite |  | orchestrates          |
  | read/  |  | API calls             |
  | write  |  | + DB writes           |
  +--------+  +------+--------+------+
                     |
                     v
              +------+----------+
              | setlistfm_client|
              +-----------------+
              | HTTP to         |
              | api.setlist.fm  |
              +-----------------+
```

### Key Design Decisions

1. **Server-rendered HTML with optional HTMX** -- No SPA. Pages render on the
   server. HTMX can be added later for inline editing and search-as-you-type
   without a full framework.

2. **No ORM** -- With only 6 tables and simple queries, raw SQL is clearer and
   has zero abstraction cost. A thin `db.py` module provides `query()`,
   `execute()`, and `fetch_one()` helpers.

3. **Import as a separate workflow** -- The import process (user input ->
   API search -> review matches -> save) is its own multi-step flow, not mixed
   into the browse/edit views.

4. **FTS5 for search** -- SQLite's built-in full-text search is fast enough
   for tens of thousands of songs and avoids pulling in Elasticsearch or similar.

5. **Rate limiter in the API client** -- `time.sleep(1.0)` between requests
   (1 req/s) with a daily counter capped at 1,440 calls/day.

---

## 6. setlist.fm API Integration

### Authentication

- Register at https://www.setlist.fm/settings/apps to get a free API key
- Every request must include the header: `x-api-key: <YOUR_KEY>`
- Also include `Accept: application/json` (default is XML)

### Rate Limiting Strategy

| Parameter | Value |
|-----------|-------|
| Per-second limit | 1 req/s |
| Daily limit | 1,440 calls/day |
| Implementation | `time.sleep(1.0)` between calls in the importer loop |
| Daily tracking | Counter reset at midnight; abort imports if approaching limit |
| Retry on 429 | Exponential backoff: 2s, 4s, 8s, max 3 retries |

### Endpoints Used

#### Primary: Search for setlists

```
GET https://api.setlist.fm/rest/1.0/search/setlists
```

**Query parameters we use:**
- `date` -- event date in `dd-MM-yyyy` format
- `venueName` -- venue name (fuzzy matched by the API)
- `artistName` -- artist name (optional, for disambiguation)
- `cityName` -- city name (optional, for disambiguation)
- `p` -- page number (results paginated at 20/page)

**Response structure (JSON):**
```json
{
  "type": "setlists",
  "itemsPerPage": 20,
  "page": 1,
  "total": 2,
  "setlist": [
    {
      "id": "abc123",
      "versionId": "def456",
      "eventDate": "01-07-2023",
      "lastUpdated": "2023-07-02T10:30:00.000+0000",
      "artist": {
        "mbid": "b10bbbfc-...",
        "name": "Radiohead",
        "sortName": "Radiohead",
        "url": "https://www.setlist.fm/setlists/radiohead-..."
      },
      "venue": {
        "id": "venue-id",
        "name": "Madison Square Garden",
        "city": {
          "id": "5128581",
          "name": "New York",
          "state": "New York",
          "stateCode": "NY",
          "coords": { "lat": 40.7128, "long": -74.006 },
          "country": { "code": "US", "name": "United States" }
        },
        "url": "https://www.setlist.fm/venue/..."
      },
      "tour": { "name": "In Rainbows Tour" },
      "sets": {
        "set": [
          {
            "name": "",
            "encore": 0,
            "song": [
              { "name": "15 Step", "info": "" },
              { "name": "There, There" },
              { "name": "Creep", "cover": { "mbid": "...", "name": "..." } }
            ]
          },
          {
            "encore": 1,
            "song": [
              { "name": "Karma Police" }
            ]
          }
        ]
      },
      "url": "https://www.setlist.fm/setlist/radiohead/...",
      "info": "Special guest appearance by..."
    }
  ]
}
```

#### Secondary: Get a specific setlist by ID

```
GET https://api.setlist.fm/rest/1.0/setlist/{setlistId}
```

Used when the user selects a specific match from search results.

#### Auxiliary: Search venues

```
GET https://api.setlist.fm/rest/1.0/search/venues
```

Used to help disambiguate venue names during import.

#### Auxiliary: Search artists

```
GET https://api.setlist.fm/rest/1.0/search/artists
```

Used if we need to resolve an artist name to an MBID.

### Import Workflow (Detailed)

```
User provides:               System does:
+---------------------+      +----------------------------------+
| Date: 2023-07-01    | ---> | 1. Parse date -> dd-MM-yyyy      |
| Venue: MSG           |      | 2. GET /search/setlists          |
+---------------------+      |    ?date=01-07-2023              |
                              |    &venueName=MSG                |
                              | 3. Receive N results             |
                              | 4. If 1 result -> auto-match     |
                              |    If N > 1  -> show review page |
                              |    If 0      -> try broader      |
                              |               search (date only) |
                              |               or mark not_found  |
                              | 5. User confirms match           |
                              | 6. Upsert artist, venue, concert,|
                              |    sets, songs into DB           |
                              | 7. Update FTS index              |
                              | 8. Update import_job status      |
                              +----------------------------------+
```

### Handling API Quirks

1. **Date format**: setlist.fm uses `dd-MM-yyyy`, not ISO 8601. Convert on
   input and store as `YYYY-MM-DD` internally.

2. **Empty setlists**: Some concerts on setlist.fm have no song data (the
   `sets.set` array is empty or missing). We still store the concert but
   display a "No setlist available" message.

3. **Multiple artists on same date/venue**: Festivals or multi-band shows
   return multiple setlist objects. Each is a separate `concert` row linked to
   its own artist.

4. **Cover/guest artist references**: The `cover` and `with` fields in songs
   reference artist objects. We upsert these into the `artists` table and
   link via `cover_artist_id` / `with_artist_id`.

5. **Attribution requirement**: setlist.fm requires linking back to them. The
   `setlistfm_url` field stores this, and the UI always displays the link.

---

## 7. UI Layout / Key Screens

### Navigation Bar (all pages)

```
+---------------------------------------------------------------+
|  Setlist Collection   |  Browse  |  Search  |  Import  |      |
+---------------------------------------------------------------+
```

### Screen 1: Browse Concerts (Home)

The default view. Shows all concerts in reverse chronological order.

```
+---------------------------------------------------------------+
|  My Setlist Collection                              [Import]   |
+---------------------------------------------------------------+
|  Filter: [Date range ▼]  [Artist ▼]  [Venue ▼]               |
+---------------------------------------------------------------+
|                                                                |
|  July 1, 2023 -- Madison Square Garden, New York              |
|  Radiohead -- In Rainbows Tour                                |
|  14 songs                                          [View ->]  |
|  ------------------------------------------------------------ |
|  March 15, 2023 -- The Fillmore, San Francisco                |
|  Phoebe Bridgers                                              |
|  18 songs                                          [View ->]  |
|  ------------------------------------------------------------ |
|  ...                                                           |
|                                                                |
|  Showing 1-20 of 47                    [< Prev]  [Next >]     |
+---------------------------------------------------------------+
```

### Screen 2: Concert Detail / Setlist View

```
+---------------------------------------------------------------+
|  <- Back to Browse                                   [Edit]    |
+---------------------------------------------------------------+
|  RADIOHEAD                                                     |
|  July 1, 2023 -- Madison Square Garden, New York              |
|  Tour: In Rainbows Tour                                       |
|  View on setlist.fm: https://www.setlist.fm/setlist/...       |
+---------------------------------------------------------------+
|                                                                |
|  SET 1                                                        |
|   1. 15 Step                                                  |
|   2. There, There                                             |
|   3. Nude                                                     |
|   4. All I Need                                               |
|   5. Weird Fishes/Arpeggi                                    |
|   ...                                                          |
|                                                                |
|  ENCORE                                                       |
|   1. Karma Police                                             |
|   2. Everything in Its Right Place                            |
|                                                                |
+---------------------------------------------------------------+
|  NOTES                                                        |
|  "Amazing show, Thom came into the crowd during Idioteque"    |
|                                                    [Edit note] |
+---------------------------------------------------------------+
```

### Screen 3: Search

```
+---------------------------------------------------------------+
|  Search                                                        |
+---------------------------------------------------------------+
|  [ Search songs, artists, venues...            ]  [Search]    |
+---------------------------------------------------------------+
|                                                                |
|  Results for "Karma Police":                                  |
|                                                                |
|  Songs (3 matches):                                           |
|   - Karma Police -- Radiohead @ MSG, July 1 2023             |
|   - Karma Police -- Radiohead @ Glastonbury, June 23 2017    |
|   - Karma Police -- Easy Star All-Stars @ Brooklyn Bowl 2019 |
|                                                                |
|  Artists (1 match):                                           |
|   - Radiohead (3 concerts)                                    |
|                                                                |
+---------------------------------------------------------------+
```

### Screen 4: Import

```
+---------------------------------------------------------------+
|  Import Setlists                                               |
+---------------------------------------------------------------+
|                                                                |
|  Enter concert dates and venues (one per line):               |
|  Format: YYYY-MM-DD, Venue Name [, Artist Name]              |
|                                                                |
|  +-----------------------------------------------------------+|
|  | 2023-07-01, Madison Square Garden                         ||
|  | 2023-03-15, The Fillmore, Phoebe Bridgers                ||
|  | 2022-11-20, Red Rocks Amphitheatre                        ||
|  +-----------------------------------------------------------+|
|                                                                |
|                                                    [Import]    |
+---------------------------------------------------------------+
```

After clicking Import, the review screen:

```
+---------------------------------------------------------------+
|  Review Matches                                                |
+---------------------------------------------------------------+
|                                                                |
|  [x] 2023-07-01, MSG                                         |
|      MATCHED: Radiohead @ Madison Square Garden (14 songs)    |
|                                                                |
|  [x] 2023-03-15, The Fillmore                                |
|      MATCHED: Phoebe Bridgers @ The Fillmore (18 songs)      |
|                                                                |
|  [!] 2022-11-20, Red Rocks                                   |
|      MULTIPLE MATCHES:                                        |
|      ( ) Foo Fighters @ Red Rocks Amphitheatre (22 songs)     |
|      ( ) Nathaniel Rateliff @ Red Rocks (19 songs)            |
|      Select one or skip                                       |
|                                                                |
|                                         [Save Selected]       |
+---------------------------------------------------------------+
```

### Screen 5: Edit Concert

```
+---------------------------------------------------------------+
|  Edit: Radiohead @ MSG -- July 1, 2023            [Save]      |
+---------------------------------------------------------------+
|                                                                |
|  Tour: [In Rainbows Tour          ]                           |
|                                                                |
|  SET 1                                        [+ Add Song]   |
|   1. [15 Step               ] [info:          ] [x]          |
|   2. [There, There          ] [info:          ] [x]          |
|   3. [Nude                  ] [info: acoustic ] [x]          |
|   ...                                                          |
|                                                                |
|  ENCORE                                       [+ Add Song]   |
|   1. [Karma Police          ] [info:          ] [x]          |
|                                                                |
|  Notes:                                                       |
|  +-----------------------------------------------------------+|
|  | Amazing show, Thom came into the crowd during Idioteque   ||
|  +-----------------------------------------------------------+|
|                                                                |
+---------------------------------------------------------------+
```

---

## 8. Security Design

This is a personal, locally-run tool. Security is minimal but sensible:

| Concern | Mitigation |
|---------|------------|
| **API key exposure** | Stored in `.env`, loaded at runtime, `.env` in `.gitignore` |
| **SQL injection** | All queries use parameterized statements (`?` placeholders), never string interpolation |
| **XSS** | Jinja2 auto-escapes all template variables by default |
| **CSRF** | Flask-WTF CSRF protection on all forms (or skip since it is localhost-only) |
| **Network exposure** | Flask dev server binds to `127.0.0.1` only -- not accessible from network |
| **Dependency supply chain** | Minimal dependencies (Flask, httpx, python-dotenv); pin versions in `requirements.txt` |
| **Database backup** | SQLite file can be copied; add a simple `make backup` or script |

---

## 9. Testing Strategy

| Layer | Tool | What to Test |
|-------|------|-------------|
| **API Client** | pytest + `respx` (httpx mock) | Mock setlist.fm responses; verify parsing of all fields, edge cases (empty setlists, missing fields, 429 retries) |
| **Database** | pytest + in-memory SQLite | Schema creation, CRUD operations, FTS search, cascading deletes |
| **Importer** | pytest | End-to-end import flow with mocked API client; match logic, conflict resolution |
| **Routes** | Flask test client | HTTP status codes, template rendering, form submissions |
| **Manual** | Browser | UI layout, search feel, edit workflow |

Key test scenarios:
- Import a date/venue that returns exactly 1 match
- Import a date/venue that returns multiple matches (festival)
- Import a date/venue that returns 0 matches
- API returns a setlist with no songs
- API returns a song that is a cover
- API returns a song with a guest artist
- API rate limit (429) triggers retry
- Editing a song name persists correctly
- FTS search finds partial matches
- Deleting a concert cascades to sets and songs

---

## 10. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| setlist.fm API changes or deprecates endpoints | Low | High | Pin to v1.0; isolate all API calls in `setlistfm_client.py` so changes are localized |
| API key gets revoked or rate-limited | Low | Medium | Implement retry logic; cache responses; all data works offline once fetched |
| Venue name matching is fuzzy / ambiguous | Medium | Medium | The review step lets the user pick the right match; also allow searching by artist name to narrow results |
| setlist.fm has no data for a concert | Medium | Low | Mark as `not_found` in import_jobs; user can manually create the concert and enter songs |
| SQLite FTS index gets out of sync | Low | Low | Rebuild FTS index on startup or via a management command |
| User loses the SQLite file | Low | High | Document backup procedure; the file is in a known location |

---

## 11. Implementation Plan

### Phase 1: Foundation (Day 1-2)

**Goal:** Flask app runs, database exists, can manually view a hardcoded concert.

- [ ] Set up project structure (directories, `requirements.txt`, `run.py`)
- [ ] Implement `config.py` (load `.env`)
- [ ] Implement `db.py` (connection management, schema creation on first run)
- [ ] Run schema migration to create all tables
- [ ] Create `base.html` template with nav bar and Pico CSS
- [ ] Create the Browse page (`concerts/index.html`) -- empty state
- [ ] Create the Detail page (`concerts/detail.html`) -- hardcoded test data
- [ ] Write tests for `db.py`

### Phase 2: API Integration (Day 3-4)

**Goal:** Can query setlist.fm and see real data.

- [ ] Implement `setlistfm_client.py`:
  - `search_setlists(date, venue_name, artist_name=None) -> list[dict]`
  - `get_setlist(setlist_id) -> dict`
  - `search_venues(name) -> list[dict]`
  - Rate limiting (sleep between requests)
  - Retry logic for 429 responses
  - Response parsing and normalization
- [ ] Write tests with mocked HTTP responses
- [ ] Manual test: query a known concert and print the result

### Phase 3: Import Workflow (Day 5-7)

**Goal:** User can paste dates/venues and get setlists saved to the database.

- [ ] Implement `importer.py`:
  - Parse user input (date, venue, optional artist)
  - Call `setlistfm_client.search_setlists()`
  - Handle single match / multiple matches / no match
  - Upsert artist, venue, concert, sets, songs
  - Update FTS index
  - Track status in `import_jobs`
- [ ] Create Import form page
- [ ] Create Review/confirm page
- [ ] Create Import status/results page
- [ ] Write tests for importer logic

### Phase 4: Browse and View (Day 8-9)

**Goal:** Browse all concerts and view full setlists.

- [ ] Wire up Browse page to real database queries
- [ ] Implement sorting (by date, artist)
- [ ] Implement pagination
- [ ] Wire up Detail page to real data
- [ ] Display sets and songs in order
- [ ] Show attribution link to setlist.fm
- [ ] Display cover/guest artist info on songs

### Phase 5: Search (Day 10-11)

**Goal:** Full-text search across all data.

- [ ] Implement `search.py` (FTS5 queries)
- [ ] Build FTS index population (on import + rebuild command)
- [ ] Create Search page with results grouped by type
- [ ] Add filter dropdowns to Browse page (artist, venue, date range)

### Phase 6: Edit (Day 12-13)

**Goal:** User can correct errors and add notes.

- [ ] Create Edit page for concerts
- [ ] Implement song editing (rename, reorder, add, delete)
- [ ] Implement notes editing on concerts and songs
- [ ] Update FTS index on edits
- [ ] Add ability to manually create a concert (for shows not on setlist.fm)

### Phase 7: Polish (Day 14)

**Goal:** Clean up rough edges.

- [ ] Error handling and user-friendly error pages
- [ ] Empty states (no concerts yet, no search results)
- [ ] Keyboard shortcuts (if desired)
- [ ] Database backup script
- [ ] Update README with setup and usage instructions
- [ ] Final round of testing

---

## 12. Dependencies (`requirements.txt`)

```
flask==3.1.*
httpx==0.28.*
python-dotenv==1.1.*
```

Dev dependencies (optional `requirements-dev.txt`):
```
pytest==8.*
respx==0.22.*
```

That is the entire dependency footprint: 3 runtime packages, 2 dev packages.

---

## 13. Configuration (`.env.example`)

```bash
# setlist.fm API key -- get one at https://www.setlist.fm/settings/apps
SETLISTFM_API_KEY=your-api-key-here

# Flask settings
FLASK_SECRET_KEY=change-this-to-a-random-string
FLASK_DEBUG=1

# Database path (default: ./data/setlists.db)
DATABASE_PATH=./data/setlists.db
```

---

## 14. Key API Client Code Sketch

```python
# app/setlistfm_client.py

import datetime
import time
import httpx

class SetlistFMClient:
    BASE_URL = "https://api.setlist.fm/rest/1.0"
    MAX_DAILY_REQUESTS = 1440

    def __init__(self, api_key: str):
        self.client = httpx.Client(
            base_url=self.BASE_URL,
            headers={
                "x-api-key": api_key,
                "Accept": "application/json",
            },
            timeout=10.0,
        )
        self._last_request_time = 0.0
        self._daily_count = 0
        self._daily_count_date = datetime.date.today()

    def _check_daily_limit(self):
        """Reset counter at midnight; raise if daily limit reached."""
        today = datetime.date.today()
        if today != self._daily_count_date:
            self._daily_count = 0
            self._daily_count_date = today
        if self._daily_count >= self.MAX_DAILY_REQUESTS:
            raise Exception(
                f"Daily API limit reached ({self.MAX_DAILY_REQUESTS} calls). "
                f"Try again tomorrow."
            )

    def _throttle(self):
        """Enforce 1 request/second spacing."""
        elapsed = time.monotonic() - self._last_request_time
        if elapsed < 1.0:
            time.sleep(1.0 - elapsed)
        self._last_request_time = time.monotonic()

    def _request(self, path: str, params: dict = None) -> dict:
        self._check_daily_limit()
        self._throttle()
        for attempt in range(3):
            resp = self.client.get(path, params=params)
            if resp.status_code == 429:
                wait = 2 ** (attempt + 1)  # 2s, 4s, 8s
                time.sleep(wait)
                continue
            resp.raise_for_status()
            self._daily_count += 1
            return resp.json()
        raise Exception("Rate limited after 3 retries")

    def search_setlists(self, date: str, venue_name: str = None,
                        artist_name: str = None, page: int = 1) -> dict:
        """
        Search for setlists. Date should be in dd-MM-yyyy format.
        Returns the full API response dict with 'setlist', 'total', etc.
        """
        params = {"date": date, "p": page}
        if venue_name:
            params["venueName"] = venue_name
        if artist_name:
            params["artistName"] = artist_name
        return self._request("/search/setlists", params)

    def get_setlist(self, setlist_id: str) -> dict:
        """Get a specific setlist by its ID."""
        return self._request(f"/setlist/{setlist_id}")

    def search_venues(self, name: str, page: int = 1) -> dict:
        """Search venues by name."""
        return self._request("/search/venues", params={"name": name, "p": page})

    def search_artists(self, name: str, page: int = 1) -> dict:
        """Search artists by name."""
        return self._request("/search/artists", params={"artistName": name, "p": page})
```

---

## Appendix A: setlist.fm API Reference (Relevant Subset)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/1.0/search/setlists` | GET | Search setlists by date, venue, artist, city, etc. |
| `/1.0/setlist/{setlistId}` | GET | Get a specific setlist by ID |
| `/1.0/search/venues` | GET | Search venues by name, city, country |
| `/1.0/search/artists` | GET | Search artists by name |
| `/1.0/artist/{mbid}/setlists` | GET | Get all setlists for an artist |
| `/1.0/venue/{venueId}/setlists` | GET | Get all setlists at a venue |

**Auth:** `x-api-key` header on every request.
**Format:** `Accept: application/json` header (default is XML).
**Rate limit:** 1 req/s, 1,440 calls/day.
**Pagination:** 20 items per page; use `p` query parameter.

## Appendix B: Data Flow for a Single Import

```
User input: "2023-07-01, Madison Square Garden"
  |
  v
Parse -> date="01-07-2023", venue="Madison Square Garden"
  |
  v
GET /search/setlists?date=01-07-2023&venueName=Madison+Square+Garden
  |
  v
API returns 2 setlists (Radiohead + opening act)
  |
  v
Show review page -> user selects both
  |
  v
For each selected setlist:
  |-- Upsert artist (by mbid) -> artists table
  |-- Upsert venue (by setlistfm_id) -> venues table
  |-- Insert concert -> concerts table
  |-- For each set in setlist:
  |     |-- Insert set -> sets table
  |     |-- For each song in set:
  |           |-- If cover: upsert cover artist
  |           |-- If with: upsert guest artist
  |           |-- Insert song -> songs table
  |-- Build FTS entry -> search_index
  |-- Update import_job -> status='matched'
  v
Redirect to concert detail page
```
