# Design Document: Event Planner App

This is the architectural design document for the event planner app. It captures
every decision needed to begin implementation. The developer should read this
end-to-end, challenge anything that feels wrong, and iterate before writing code.

The product concept and feature list live in README.md. This document covers
*how* the app is built, not *what* it does.

---

## Table of Contents

1. [Technology Stack](#technology-stack)
2. [Runtime Dependencies](#runtime-dependencies)
3. [Project Structure](#project-structure)
4. [Architecture](#architecture)
5. [Database Schema](#database-schema)
6. [Route Design](#route-design)
7. [Authentication and Authorization](#authentication-and-authorization)
8. [Setlist FM Integration](#setlist-fm-integration)
9. [Calendar Features](#calendar-features)
10. [Frontend Approach](#frontend-approach)
11. [Testing Strategy](#testing-strategy)
12. [Deployment](#deployment)
13. [Implementation Roadmap](#implementation-roadmap)

---

## Technology Stack

| Layer            | Choice                        | Rationale                                                                 |
|------------------|-------------------------------|---------------------------------------------------------------------------|
| Language         | Python 3.12+                  | Sole developer knows it; rich ecosystem for web and data tasks            |
| Web Framework    | Flask                         | Lightweight, no fighting framework opinions, right-sized for solo dev     |
| Database         | SQLite (WAL mode)             | Zero infrastructure, trivial backups (`cp`), FTS5 for search             |
| Data Access      | Raw parameterized SQL         | ~10 tables; clearer than an ORM, avoids N+1 traps, repository layer isolates SQL |
| Templating       | Jinja2 (ships with Flask)     | Server-rendered HTML; no separate frontend build                          |
| Dynamic UI       | HTMX                          | Declarative AJAX from HTML attributes; no custom JS for most interactions |
| Calendar UI      | FullCalendar (JS via CDN)     | The one place a JS library earns its keep; mature, well-documented        |
| CSS              | Pico CSS (via CDN)            | Classless/minimal CSS framework; good defaults without writing CSS        |
| Auth             | Flask-Login + bcrypt           | Session-based auth; Flask-Login manages the session lifecycle             |
| HTTP Client      | httpx                         | For Setlist FM API calls; async-capable if needed later                   |
| Calendar Export  | icalendar                     | Generates .ics files; universal standard, no OAuth needed                 |
| Config           | python-dotenv                 | Loads `.env` into `os.environ`; keeps secrets out of code                 |
| Testing          | pytest + pytest-cov           | De facto standard; fixtures and parametrize cover all test patterns needed|
| CSRF Protection  | Flask-WTF (CSRFProtect only)  | Provides CSRF tokens for forms; not using WTForms form classes            |

### Why Not Django

Django's ORM, admin panel, and migration system are built for larger teams and
larger schemas. For ~10 tables with hand-written SQL, Django's conventions
become overhead. Flask lets us structure the app exactly how we want.

### Why Not Postgres

SQLite with WAL mode handles concurrent reads well and supports a single-writer
workload. The app is single-server. If it ever needs Postgres, the repository
layer isolates every SQL statement -- migration is a localized change, not a
rewrite.

---

## Runtime Dependencies

Target: 7 Python packages + 2 JS libraries via CDN.

**Python (in pyproject.toml):**

| Package       | Purpose                          |
|---------------|----------------------------------|
| flask         | Web framework                    |
| flask-login   | Session management               |
| flask-wtf     | CSRF protection                  |
| bcrypt        | Password hashing                 |
| icalendar     | .ics file generation             |
| httpx         | HTTP client for Setlist FM API   |
| python-dotenv | Environment variable loading     |

**JavaScript (loaded via CDN, no npm/node):**

| Library       | Purpose                          |
|---------------|----------------------------------|
| HTMX          | Dynamic page updates             |
| FullCalendar  | Calendar view component          |

Pico CSS is also loaded via CDN.

---

## Project Structure

```
setlist_collection_manager/
|-- app/
|   |-- __init__.py              # create_app factory
|   |-- config.py                # Configuration classes
|   |-- db.py                    # Database connection, init, teardown
|   |-- schema.sql               # Full DDL (all CREATE TABLE statements)
|   |-- blueprints/
|   |   |-- __init__.py
|   |   |-- auth.py              # Login, logout, register
|   |   |-- events.py            # Events CRUD, attendance
|   |   |-- artists.py           # Artists CRUD
|   |   |-- venues.py            # Venues CRUD
|   |   |-- shows.py             # Show Runs CRUD
|   |   |-- performances.py      # Performances CRUD, setlist linking
|   |   |-- calendar.py          # Calendar view, .ics export
|   |   |-- social.py            # Friends, following, profiles
|   |   `-- search.py            # FTS5 search endpoint
|   |-- repositories/
|   |   |-- __init__.py
|   |   |-- event_repo.py
|   |   |-- artist_repo.py
|   |   |-- venue_repo.py
|   |   |-- show_repo.py
|   |   |-- performance_repo.py
|   |   |-- user_repo.py
|   |   |-- note_repo.py
|   |   |-- attendance_repo.py
|   |   `-- relationship_repo.py
|   |-- services/
|   |   |-- __init__.py
|   |   |-- setlistfm.py         # Setlist FM API client
|   |   `-- ical.py              # .ics generation logic
|   |-- templates/
|   |   |-- base.html            # Layout with HTMX, Pico CSS, nav
|   |   |-- auth/
|   |   |   |-- login.html
|   |   |   `-- register.html
|   |   |-- events/
|   |   |   |-- list.html
|   |   |   |-- detail.html
|   |   |   `-- form.html
|   |   |-- artists/
|   |   |   |-- list.html
|   |   |   |-- detail.html
|   |   |   `-- form.html
|   |   |-- venues/
|   |   |   |-- list.html
|   |   |   |-- detail.html
|   |   |   `-- form.html
|   |   |-- calendar/
|   |   |   `-- view.html
|   |   |-- social/
|   |   |   |-- profile.html
|   |   |   `-- friends.html
|   |   `-- partials/            # HTMX partial templates
|   |       |-- attendance_button.html
|   |       |-- note_list.html
|   |       |-- search_results.html
|   |       `-- friend_overlay.html
|   `-- static/
|       |-- css/
|       |   `-- app.css          # Minimal overrides on top of Pico
|       `-- js/
|           `-- calendar.js      # FullCalendar initialization
|-- tests/
|   |-- conftest.py              # App fixture, test DB, test client
|   |-- test_auth.py
|   |-- test_events.py
|   |-- test_artists.py
|   |-- test_venues.py
|   |-- test_performances.py
|   |-- test_calendar.py
|   |-- test_social.py
|   |-- test_setlistfm.py
|   `-- repositories/
|       |-- test_event_repo.py
|       `-- ...                  # Mirror of app/repositories/
|-- scripts/
|   |-- install.sh               # First-time server setup
|   |-- update.sh                # Deploy new version
|   `-- backup.sh                # Database backup
|-- migrations/
|   |-- 001_initial_schema.sql   # Baseline (matches schema.sql)
|   `-- ...                      # Incremental schema changes
|-- passenger_wsgi.py            # WSGI entry point for shared hosting
|-- pyproject.toml               # Project metadata, dependencies, tool config
|-- .env.example                 # Template for environment variables
|-- .gitignore
|-- CLAUDE.md
|-- DESIGN.md                    # This file
`-- README.md
```

### Key Conventions

- **One blueprint per domain area.** Blueprints live in `app/blueprints/` and
  register their own routes and URL prefixes.
- **One repository per entity.** Repositories accept a database connection and
  return dicts (not objects). They contain only SQL and row-mapping logic.
- **Services hold integration logic.** The Setlist FM client and iCal generator
  live here. Anything that coordinates across repositories also goes here.
- **Partials directory for HTMX.** Templates returned by HTMX-targeted
  endpoints live in `templates/partials/` so they are visually distinct from
  full-page templates.

---

## Architecture

The app follows a layered structure within a single Flask process:

```
HTTP Request
    |
    v
[Blueprint Route]  -- validates input, calls repository/service, renders template
    |
    v
[Service Layer]    -- optional; used for cross-cutting logic (setlist FM, ical)
    |
    v
[Repository Layer] -- executes parameterized SQL, returns dicts
    |
    v
[SQLite via db.py] -- connection-per-request, WAL mode
```

### Rules

1. **Blueprints never write SQL.** They call repositories or services.
2. **Repositories never import Flask.** They receive a `sqlite3.Connection` as
   an argument. This makes them testable with a plain in-memory database.
3. **Services may call repositories** but repositories never call services.
4. **No god objects.** If a function needs data from two repositories, it lives
   in the blueprint or a service -- not crammed into one repository.

### Database Connection Lifecycle

Flask's `g` object holds the connection for the duration of a request.
`db.py` provides:

- `get_db()` -- returns the connection for the current request, creating it if
  needed. Sets WAL mode and enables foreign keys on first use.
- `close_db()` -- registered as a teardown function; closes the connection after
  each request.
- `init_db()` -- reads `schema.sql` and executes it. Called once during setup or
  via a CLI command (`flask init-db`).

```python
# app/db.py sketch
import sqlite3
from flask import g, current_app

def get_db():
    if "db" not in g:
        g.db = sqlite3.connect(current_app.config["DATABASE"])
        g.db.row_factory = sqlite3.Row
        g.db.execute("PRAGMA journal_mode=WAL")
        g.db.execute("PRAGMA foreign_keys=ON")
    return g.db

def close_db(e=None):
    db = g.pop("db", None)
    if db is not None:
        db.close()
```

---

## Database Schema

### Design Decisions

1. **Event-to-Performance is one-to-many.** An Event has many Performances,
   but each Performance belongs to exactly one Event. A simple `event_id` FK
   on the `performances` table — no junction table needed.

2. **Polymorphic Notes.** A single `notes` table uses `note_type` (enum of
   `event`, `performance`, `artist`, `venue`) and `target_id` to point at
   any entity. No referential integrity via FK on `target_id` -- the
   application enforces validity. This avoids four nullable FK columns and
   scales to new entity types without schema changes.

3. **Planned vs. attended is derived from date, not stored.** If the
   performance date is in the future, the user plans to attend. If it's in
   the past, they attended. This avoids drift. Performances have a
   `cancelled` flag (cancelled for everyone). Attendance has a `skipped`
   flag (user-specific: planned to go but didn't).

4. **Venue lives on Event, stage lives on Performance.** The Event holds the
   Venue FK (the overall location). Performances have an optional `stage`
   text field for sub-locations within the venue (e.g., "Main Stage",
   "Room B", "Balcony"). No separate Venue FK on Performance — the venue
   is always inherited from the Event.

5. **Relationships are directional rows.** User A following User B is one row.
   Mutual follows (friendship) are two rows. Blocking is a single row from
   blocker to blocked.

6. **FTS5 virtual table** for full-text search across artist names, venue
   names, and event names. Maintained manually (insert/update/delete triggers
   or application-level sync).

7. **Change history for shared entities.** Every create, update, lock, and
   unlock on artists, venues, and events is recorded in `change_history` with
   a JSON snapshot of the entity state. Admins can view the full history and
   revert to any prior state. The polymorphic pattern (`entity_type` +
   `entity_id`) mirrors the notes table approach.

8. **Admin users and entity locking.** Users have an `is_admin` flag. Admins
   can lock artists, venues, and events (preventing edits by non-admins),
   lock user accounts, view change history, and revert entities. Locked
   entities show a visual indicator and hide edit controls for non-admins.

9. **All IDs are INTEGER PRIMARY KEY** (SQLite's built-in rowid alias). No
   UUIDs -- there is no distributed system to coordinate with.

10. **Timestamps stored as UTC in ISO 8601 TEXT** (`YYYY-MM-DD` for dates,
   `YYYY-MM-DDTHH:MM:SS` for datetimes). All times are stored in UTC and
   converted to the user's timezone on display. Users set their timezone in
   their profile. SQLite's date functions work with this format and it sorts
   correctly.

### Full Schema

```sql
-- schema.sql

-- ============================================================
-- Users
-- ============================================================
CREATE TABLE IF NOT EXISTS users (
    id              INTEGER PRIMARY KEY,
    username        TEXT    NOT NULL UNIQUE,
    email           TEXT    NOT NULL UNIQUE,
    password_hash   TEXT    NOT NULL,
    email_verified  INTEGER NOT NULL DEFAULT 0,  -- boolean: email address confirmed
    approved        INTEGER NOT NULL DEFAULT 0,  -- boolean: admin-granted write access
    is_admin        INTEGER NOT NULL DEFAULT 0,  -- boolean: admin privileges
    locked          INTEGER NOT NULL DEFAULT 0,  -- boolean: account locked by admin
    timezone        TEXT    NOT NULL DEFAULT 'America/New_York',  -- IANA timezone
    setlistfm_key   TEXT,               -- per-user Setlist FM API key
    calendar_token  TEXT,               -- random token for .ics feed URL
    created_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT    NOT NULL DEFAULT (datetime('now'))
);

-- ============================================================
-- Venues
-- ============================================================
CREATE TABLE IF NOT EXISTS venues (
    id              INTEGER PRIMARY KEY,
    name            TEXT    NOT NULL,
    city            TEXT    NOT NULL,
    state           TEXT,
    street_address  TEXT,
    capacity        INTEGER,            -- seats or standing capacity
    url             TEXT,
    locked          INTEGER NOT NULL DEFAULT 0,  -- boolean: locked by admin
    created_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT    NOT NULL DEFAULT (datetime('now'))
);

-- ============================================================
-- Artists
-- ============================================================
CREATE TABLE IF NOT EXISTS artists (
    id              INTEGER PRIMARY KEY,
    name            TEXT    NOT NULL,
    artist_type     TEXT    NOT NULL CHECK (
                        artist_type IN (
                            'band', 'comedian', 'performer',
                            'troupe', 'theater_company', 'other'
                        )
                    ),
    url             TEXT,
    locked          INTEGER NOT NULL DEFAULT 0,  -- boolean: locked by admin
    created_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT    NOT NULL DEFAULT (datetime('now'))
);

-- ============================================================
-- Show Runs (tours, festival series, theatrical runs, etc.)
-- ============================================================
CREATE TABLE IF NOT EXISTS show_runs (
    id              INTEGER PRIMARY KEY,
    name            TEXT    NOT NULL,
    artist_id       INTEGER,            -- nullable: not all runs have a single artist
    created_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    FOREIGN KEY (artist_id) REFERENCES artists(id) ON DELETE SET NULL
);

-- ============================================================
-- Events (a collection of performances across dates)
-- ============================================================
CREATE TABLE IF NOT EXISTS events (
    id              INTEGER PRIMARY KEY,
    name            TEXT    NOT NULL,
    venue_id        INTEGER NOT NULL,   -- primary/overall venue
    start_date      TEXT    NOT NULL,   -- YYYY-MM-DD
    end_date        TEXT    NOT NULL,   -- YYYY-MM-DD (same as start for single-day)
    url             TEXT,
    locked          INTEGER NOT NULL DEFAULT 0,  -- boolean: locked by admin
    created_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    FOREIGN KEY (venue_id) REFERENCES venues(id) ON DELETE RESTRICT
);

-- ============================================================
-- Performances (a single showing at a single venue and time)
-- ============================================================
CREATE TABLE IF NOT EXISTS performances (
    id              INTEGER PRIMARY KEY,
    event_id        INTEGER NOT NULL,   -- each performance belongs to one event
    artist_id       INTEGER,            -- nullable: some performances have no listed artist
    stage           TEXT,               -- optional: sub-location within venue (e.g., "Main Stage")
    show_run_id     INTEGER,            -- nullable: not all performances belong to a run
    date            TEXT    NOT NULL,   -- YYYY-MM-DD
    time            TEXT,               -- HH:MM (nullable for TBD times)
    cancelled       INTEGER NOT NULL DEFAULT 0,  -- boolean: show was cancelled
    setlist_url     TEXT,               -- link to setlist.fm page
    display_order   INTEGER NOT NULL DEFAULT 0,  -- ordering within an event
    created_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    FOREIGN KEY (event_id)    REFERENCES events(id)    ON DELETE CASCADE,
    FOREIGN KEY (artist_id)   REFERENCES artists(id)   ON DELETE SET NULL,
    FOREIGN KEY (show_run_id) REFERENCES show_runs(id)  ON DELETE SET NULL
);

-- ============================================================
-- Attendance (links users to performances with status)
-- ============================================================
CREATE TABLE IF NOT EXISTS attendance (
    id              INTEGER PRIMARY KEY,
    user_id         INTEGER NOT NULL,
    performance_id  INTEGER NOT NULL,
    skipped         INTEGER NOT NULL DEFAULT 0,  -- boolean: planned to go but didn't
    created_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    UNIQUE (user_id, performance_id),
    FOREIGN KEY (user_id)        REFERENCES users(id)        ON DELETE CASCADE,
    FOREIGN KEY (performance_id) REFERENCES performances(id) ON DELETE CASCADE
);

-- ============================================================
-- Notes (polymorphic: note_type + target_id)
-- ============================================================
CREATE TABLE IF NOT EXISTS notes (
    id              INTEGER PRIMARY KEY,
    user_id         INTEGER NOT NULL,
    note_type       TEXT    NOT NULL CHECK (
                        note_type IN ('event', 'performance', 'artist', 'venue')
                    ),
    target_id       INTEGER NOT NULL,   -- FK enforced by application, not DB
    body            TEXT    NOT NULL,
    privacy         TEXT    NOT NULL DEFAULT 'private' CHECK (
                        privacy IN ('public', 'friends', 'private')
                    ),
    created_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- ============================================================
-- Relationships (directional: follower -> followed)
-- ============================================================
CREATE TABLE IF NOT EXISTS relationships (
    id              INTEGER PRIMARY KEY,
    user_id         INTEGER NOT NULL,   -- the user performing the action
    related_user_id INTEGER NOT NULL,   -- the target user
    relationship    TEXT    NOT NULL CHECK (
                        relationship IN ('following', 'blocked')
                    ),
    created_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    UNIQUE (user_id, related_user_id),
    FOREIGN KEY (user_id)         REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (related_user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- ============================================================
-- Change History (audit log for shared entities)
-- ============================================================
CREATE TABLE IF NOT EXISTS change_history (
    id              INTEGER PRIMARY KEY,
    entity_type     TEXT    NOT NULL CHECK (
                        entity_type IN ('artist', 'venue', 'event')
                    ),
    entity_id       INTEGER NOT NULL,
    user_id         INTEGER NOT NULL,   -- who made the change
    action          TEXT    NOT NULL CHECK (
                        action IN ('create', 'update', 'lock', 'unlock')
                    ),
    snapshot        TEXT    NOT NULL,   -- JSON snapshot of entity state after change
    created_at      TEXT    NOT NULL DEFAULT (datetime('now')),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
);

-- ============================================================
-- Schema Version (migration tracking)
-- ============================================================
CREATE TABLE IF NOT EXISTS schema_version (
    version     INTEGER PRIMARY KEY,
    applied_at  TEXT    NOT NULL DEFAULT (datetime('now')),
    filename    TEXT    NOT NULL
);

-- ============================================================
-- Indexes
-- ============================================================
CREATE INDEX IF NOT EXISTS idx_performances_event      ON performances(event_id);
CREATE INDEX IF NOT EXISTS idx_performances_date       ON performances(date);
CREATE INDEX IF NOT EXISTS idx_performances_artist     ON performances(artist_id);
CREATE INDEX IF NOT EXISTS idx_performances_show_run   ON performances(show_run_id);
CREATE INDEX IF NOT EXISTS idx_events_start_date       ON events(start_date);
CREATE INDEX IF NOT EXISTS idx_events_venue            ON events(venue_id);
CREATE INDEX IF NOT EXISTS idx_attendance_user         ON attendance(user_id);
CREATE INDEX IF NOT EXISTS idx_attendance_performance  ON attendance(performance_id);
CREATE INDEX IF NOT EXISTS idx_performances_cancelled   ON performances(cancelled);
CREATE INDEX IF NOT EXISTS idx_notes_target            ON notes(note_type, target_id);
CREATE INDEX IF NOT EXISTS idx_notes_user              ON notes(user_id);
CREATE INDEX IF NOT EXISTS idx_relationships_user      ON relationships(user_id);
CREATE INDEX IF NOT EXISTS idx_relationships_related   ON relationships(related_user_id);
CREATE INDEX IF NOT EXISTS idx_change_history_entity   ON change_history(entity_type, entity_id);
CREATE INDEX IF NOT EXISTS idx_change_history_user     ON change_history(user_id);

-- ============================================================
-- Full-Text Search (FTS5)
-- ============================================================
CREATE VIRTUAL TABLE IF NOT EXISTS search_index USING fts5(
    name,
    entity_type,        -- 'artist', 'venue', 'event'
    entity_id UNINDEXED -- not searchable, just carried along for lookups
);
```

### Entity Relationship Summary

```
Users ----< Attendance >---- Performances
Users ----< Notes
Users ----< Relationships >---- Users

Events ----< Performances

Performances >---- Artists
Performances >---- Show Runs

Show Runs >---- Artists

Events >---- Venues
```

### Notes on the Schema

**Why `ON DELETE RESTRICT` for Venues?** Deleting a venue that has events or
performances attached is almost certainly a mistake. Force the user to reassign
or delete dependents first.

**Why `ON DELETE SET NULL` for Artists on Performances?** An artist being
removed from the system should not destroy performance records. The performance
still happened; it just loses its artist link.

**Why no `ON DELETE` enforcement on `notes.target_id`?** The polymorphic
`target_id` cannot have a real FK constraint since it points at different tables
depending on `note_type`. The application must handle orphan cleanup -- either
eagerly on entity deletion or lazily on note retrieval.

**FTS5 sync strategy.** The `search_index` table is maintained by the
application, not triggers. When an artist, venue, or event is created, updated,
or deleted, the corresponding repository function also updates the search index.
This keeps the sync logic visible and testable rather than hidden in triggers.

---

## Route Design

The app serves HTML pages. HTMX handles partial-page updates for interactive
elements. This is not a REST API -- URLs return rendered HTML.

### URL Structure

| Blueprint       | Prefix           | Key Routes                                                |
|-----------------|------------------|-----------------------------------------------------------|
| `auth`          | `/auth`          | `GET /login`, `POST /login`, `GET /register`, `POST /register`, `POST /logout`, `GET /verify/<token>`, `POST /resend-verification` |
| `events`        | `/events`        | `GET /` (list), `GET /new`, `POST /new`, `GET /<id>`, `GET /<id>/edit`, `POST /<id>/edit`, `POST /<id>/delete` |
| `artists`       | `/artists`       | Same CRUD pattern as events                               |
| `venues`        | `/venues`        | Same CRUD pattern as events                               |
| `shows`         | `/shows`         | Same CRUD pattern as events (for show runs)               |
| `performances`  | `/performances`  | `GET /` (list), `GET /new`, `POST /new`, `GET /<id>`, `POST /<id>/attend`, `POST /<id>/unattend` |
| `calendar`      | `/calendar`      | `GET /` (calendar view), `GET /feed.ics` (iCal export), `GET /data.json` (FullCalendar JSON feed) |
| `social`        | `/social`        | `GET /profile/<username>`, `POST /follow/<user_id>`, `POST /unfollow/<user_id>`, `POST /block/<user_id>`, `GET /friends` |
| `search`        | `/search`        | `GET /` (search page), `GET /results` (HTMX partial)     |
| `admin`         | `/admin`         | `GET /users` (pending/all users list), `POST /users/<id>/approve` |

The root URL (`/`) shows the home page: a list of the user's recent and
upcoming events (by attendance). Unauthenticated users are redirected to login.

### HTMX Endpoints

HTMX endpoints return HTML fragments, not full pages. They are identified by
checking the `HX-Request` header. Key HTMX interactions:

| Interaction                  | Trigger                 | Returns                         |
|------------------------------|-------------------------|---------------------------------|
| Mark attendance              | Button click            | Updated attendance button state |
| Add/edit note                | Form submit in modal    | Updated note list               |
| Search-as-you-type           | Input keyup (debounced) | Search results partial          |
| Entity autocomplete          | Input keyup (debounced) | Dropdown of Setlist FM suggestions |
| Toggle friend on calendar    | Checkbox change         | Updated calendar event data     |
| Load more items (pagination) | Scroll / click "more"   | Next page of list items         |

### Pagination

List pages use offset-based pagination (good enough for the data volumes here).
HTMX's `hx-trigger="revealed"` on a sentinel element at the bottom of the list
enables infinite scroll with a "Load more" fallback link for non-JS users.

---

## Authentication and Authorization

### Authentication Flow

1. **Registration**: Username + email + password. Password hashed with bcrypt
   (via the `bcrypt` library directly, cost factor 12).
2. **Email verification**: On registration, the account is created with
   `email_verified = 0`. A verification email is sent containing a signed,
   time-limited token (using `itsdangerous`, already a Flask dependency).
   Clicking the link sets `email_verified = 1`. A "resend verification"
   option is available on the home page for unverified users.
3. **Login**: Username + password. Flask-Login manages the session cookie.
4. **Session**: Server-side session stored in Flask's default signed cookie.
   `SESSION_COOKIE_HTTPONLY=True`, `SESSION_COOKIE_SAMESITE='Lax'`.
5. **Logout**: Clears the session.

### Authorization Rules

- **Public pages (no login required)**: Login, register, public artist/venue
  listings, public notes.
- **Read-only (logged in, default for new accounts)**: All new accounts start
  read-only (`approved = 0`). Read-only users can browse all public listings,
  view event/artist/venue detail pages, and manage their own profile settings.
  They cannot create or edit entities, track attendance, add notes, or use
  social features. The UI hides write controls and shows a banner explaining
  the account is pending approval.
- **Approved (full access)**: An admin sets `approved = 1` to grant write
  access. Approved users can create/edit entities, track attendance, add
  notes, use calendar and social features. Email verification is still
  required independently -- an approved but unverified user is prompted to
  verify before gaining full access.
- **Ownership**: Users can only edit/delete their own attendance records, notes,
  and profile. Entity data (artists, venues, events) is shared -- any approved
  user can edit, unless the entity is locked.
- **Admin**: Users with `is_admin` set can: approve new accounts, view change
  history for any entity, revert an entity to a previous state from its
  history, lock/unlock artists, venues, and events to prevent edits, and
  lock/unlock user accounts.
- **Note privacy**: Private notes visible only to author. Friends-only notes
  visible to author and mutual follows. Public notes visible to everyone.
- **Blocking**: A blocked user cannot see the blocker's profile, notes, or
  attendance. The block is silent -- no error message, just invisible data.

### CSRF Protection

Flask-WTF's `CSRFProtect` is initialized app-wide. Every form includes
`{{ csrf_token() }}`. HTMX requests include the CSRF token via a meta tag and
`hx-headers` configuration in the base template:

```html
<meta name="csrf-token" content="{{ csrf_token() }}">
<body hx-headers='{"X-CSRFToken": "{{ csrf_token() }}"}'>
```

### Password Storage

- Bcrypt with cost factor 12 (default).
- Passwords validated on registration: minimum 8 characters, no maximum
  (bcrypt truncates at 72 bytes -- document this in the UI).
- Password reset via email planned for Phase 5 (Polish) or earlier.

---

## Setlist FM Integration

### Overview

Users optionally store their Setlist FM API key in their profile. The app uses
it for two purposes:

1. **Entity autocomplete** — when adding a venue, artist, or tour, a
   search-as-you-type field queries the Setlist FM API and offers suggestions.
   Selecting a suggestion pre-fills the form fields from the API data.
2. **Setlist linking** — when adding a performance, the user can search for and
   link a setlist.

The app links to setlist.fm pages rather than storing setlist data locally
(respects their terms and avoids stale data).

### Client Design

`app/services/setlistfm.py` contains a single `SetlistFMClient` class:

```python
class SetlistFMClient:
    BASE_URL = "https://api.setlist.fm/rest/1.0"

    def __init__(self, api_key: str):
        self.client = httpx.Client(
            base_url=self.BASE_URL,
            headers={
                "x-api-key": api_key,
                "Accept": "application/json",
            },
            timeout=10.0,
        )

    def search_artists(self, name: str) -> dict:
        """Search for artists by name."""
        return self._get("/search/artists", params={"artistName": name})

    def search_venues(self, name: str, city: str | None = None) -> dict:
        """Search for venues by name, optionally filtered by city."""
        params = {"name": name}
        if city:
            params["cityName"] = city
        return self._get("/search/venues", params=params)

    def search_setlists(self, artist_name: str, date: str | None = None) -> dict:
        """Search for setlists. Date format: dd-MM-yyyy (setlist.fm quirk)."""
        params = {"artistName": artist_name}
        if date:
            params["date"] = date
        return self._get("/search/setlists", params=params)

    def get_setlist(self, setlist_id: str) -> dict:
        """Get a single setlist by ID."""
        return self._get(f"/setlist/{setlist_id}")

    def _get(self, path: str, params: dict | None = None) -> dict:
        """Make a GET request with rate limiting and error handling."""
        response = self.client.get(path, params=params)
        response.raise_for_status()
        return response.json()
```

### Rate Limiting

- Setlist FM allows 16 req/s burst, but we target 2 req/s to be conservative.
- Rate limiting is implemented in the client via a simple `time.sleep()` guard,
  not in the calling code.
- 429 responses trigger exponential backoff: 1s, 2s, 4s, then fail.

### Graceful Degradation

- If a user has no API key configured, setlist search UI elements are hidden.
- If the API is unreachable or returns errors, the UI shows a message and the
  performance can still be saved without a setlist link.
- API errors never break the page or prevent form submission.

### Attribution

Setlist FM requires attribution when displaying data sourced from their API
(e.g., autocomplete suggestions, pre-filled form fields). Pages that simply
link to setlist.fm do not require attribution. Attribution text: "Data provided
by setlist.fm" with a link to setlist.fm.

---

## Calendar Features

### Calendar View (FullCalendar)

The `/calendar` page loads FullCalendar from CDN. It fetches event data from
`/calendar/data.json`, which returns the logged-in user's attendance records
formatted as FullCalendar event objects.

```json
[
    {
        "id": "performance-42",
        "title": "Radiohead @ Madison Square Garden",
        "start": "2026-06-15T20:00:00",
        "url": "/performances/42",
        "color": "#4a90d9",
        "extendedProps": {
            "status": "planned",
            "owner": "self"
        }
    }
]
```

### Friend Overlays

Checkboxes on the calendar page toggle friends' events on and off. Each toggle
fires an HTMX request to `/calendar/data.json?include=<user_id>` which returns
the friend's public attendance data merged with the user's own data.
FullCalendar re-renders with the combined dataset. Friend events use a
different color.

### .ics Export

`GET /calendar/feed.ics` returns an iCalendar file containing the user's
attendance records. This URL is stable and unauthenticated (uses a per-user
token in the query string, e.g., `/calendar/feed.ics?token=<random-token>`).
External calendar apps poll this URL to stay synced.

The token is generated on first export request and stored in the `users` table
(add a `calendar_token` column). Users can regenerate it from their profile to
revoke old subscriptions.

The `calendar_token` column is included in the `users` table in the schema
above.

### iCal Event Format

Each attendance record becomes a `VEVENT`:

```
BEGIN:VEVENT
SUMMARY:Radiohead @ Madison Square Garden
DTSTART:20260615T200000
LOCATION:Madison Square Garden, New York
URL:https://yourapp.example/performances/42
STATUS:TENTATIVE
END:VEVENT
```

Status mapping: future date → `TENTATIVE`, past date → `CONFIRMED`,
performance `cancelled` → `CANCELLED`. Skipped attendance records are omitted
from the feed.

---

## Frontend Approach

### Principles

1. **Server renders full pages.** Every URL returns a complete HTML document.
   The app works without JavaScript (degraded but functional).
2. **HTMX handles interactivity.** Attendance toggles, search, pagination,
   notes, and friend overlays use HTMX for partial updates.
3. **Pico CSS provides styling.** Semantic HTML elements get styled
   automatically. Minimal custom CSS in `app.css`.
4. **FullCalendar is the exception.** The calendar page requires JavaScript.
   It is the only page that does not work without JS.

### Base Template Structure

```html
<!DOCTYPE html>
<html lang="en" data-theme="light">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{% block title %}Events{% endblock %}</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css">
    <link rel="stylesheet" href="{{ url_for('static', filename='css/app.css') }}">
    <meta name="csrf-token" content="{{ csrf_token() }}">
</head>
<body hx-headers='{"X-CSRFToken": "{{ csrf_token() }}"}'>
    <nav class="container">
        <!-- App name, nav links, login/logout -->
    </nav>
    <main class="container">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% for category, message in messages %}
                <div role="alert" class="{{ category }}">{{ message }}</div>
            {% endfor %}
        {% endwith %}
        {% block content %}{% endblock %}
    </main>
    <script src="https://unpkg.com/htmx.org@2"></script>
    {% block scripts %}{% endblock %}
</body>
</html>
```

### HTMX Patterns Used

| Pattern                  | HTMX Attributes                                           | Example Use                  |
|--------------------------|-----------------------------------------------------------|------------------------------|
| Click to update          | `hx-post`, `hx-target`, `hx-swap="outerHTML"`            | Attendance toggle button     |
| Search as you type       | `hx-get`, `hx-trigger="keyup changed delay:300ms"`       | Search bar                   |
| Autocomplete from API    | `hx-get`, `hx-trigger="keyup changed delay:300ms"`, `hx-target` dropdown | Artist/venue/tour form fields |
| Infinite scroll          | `hx-get`, `hx-trigger="revealed"`, `hx-swap="afterend"`  | List pagination              |
| Inline edit              | `hx-get` (load form), `hx-post` (save)                   | Notes editing                |
| Confirm before action    | `hx-confirm="Are you sure?"`                              | Delete actions               |

---

## Testing Strategy

### Test Pyramid

```
         /  E2E  \           -- manual or optional Playwright later
        /----------\
       / Integration \       -- Flask test client, real SQLite (in-memory)
      /----------------\
     /    Unit Tests     \   -- Repositories, services, helpers
    /______________________\
```

### Test Infrastructure

- **Database**: In-memory SQLite (`:memory:`) for all tests. `conftest.py`
  provides a fixture that creates the schema and optionally seeds test data.
- **Flask test client**: `app.test_client()` for route tests. No running server.
- **Setlist FM mocking**: `respx` library to mock `httpx` calls. Test data
  fixtures with realistic API responses.
- **No external dependencies in tests.** All network calls are mocked.

### What Gets Tested

| Layer          | What to Test                                          | How                                 |
|----------------|-------------------------------------------------------|--------------------------------------|
| Repositories   | SQL correctness, edge cases, constraint violations    | Direct function calls with in-memory DB |
| Services       | Setlist FM client, iCal generation, business logic    | Unit tests with mocked dependencies |
| Blueprints     | Route access control, form validation, response codes | Flask test client                   |
| Auth           | Login, logout, registration, CSRF, session behavior   | Flask test client                   |
| Notes privacy  | Visibility rules for public/friends/private           | Integration tests with multiple users|

### Coverage Target

Aim for 80%+ line coverage. The repositories and auth flows should be near
100%. Template rendering correctness is verified by checking for key strings in
response data -- no DOM parsing needed.

### Test Configuration

`pyproject.toml` configures pytest:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=app --cov-report=term-missing"
```

---

## Deployment

### Target Environment

Shared hosting environment (e.g., DreamHost). The app runs under a user
account's domain subdirectory, not as root on a dedicated server. No sudo
access, no systemd, no containers.

### Separation of Repo and Deployed Instance

The git repo is for development only. The app is **never served directly from
the repo**. Instead, an installer script deploys the app to a target directory
(e.g., `/opt/eventplanner/`). This keeps the deployment clean of git history,
test files, design docs, and development tooling.

```
Development:  ~/repos/setlist_collection_manager/          (git repo, local machine)
Production:   ~/example.com/eventplanner/                  (deployed instance, shared host)
```

The exact path depends on the hosting provider. Shared hosts typically map
domains to directories under the user's home (e.g., `~/example.com/`). The
app lives in a subdirectory within that.

### Deployed Directory Layout

```
~/example.com/eventplanner/
|-- app/                    # Application code (current deploy)
|-- migrations/             # SQL migration scripts (current deploy)
|-- previous/               # Snapshot of prior deploy (for rollback)
|   |-- app/
|   `-- migrations/
|-- instance/
|   |-- app.db              # SQLite database (created on install, persisted)
|   `-- uploads/            # Future: user uploads
|-- .env                    # Environment config (created on install, persisted)
|-- venv/                   # Python virtualenv (created on install)
`-- logs/                   # Application logs
```

### Installer / Updater

The repo includes a `scripts/` directory with deployment tooling:

```
scripts/
|-- install.sh              # First-time setup on a fresh server
|-- update.sh               # Deploy new version from repo
|-- rollback.sh             # Revert to previous deploy
`-- backup.sh               # Database backup
```

**`install.sh`** (first-time setup):
1. Accepts target directory as an argument (e.g.,
   `./install.sh ~/example.com/eventplanner`)
2. Creates the target directory structure
3. Copies `app/` and `migrations/` to the target
4. Creates a Python virtualenv and installs dependencies from `pyproject.toml`
5. Copies `.env.example` to the target as `.env` and prompts for secrets
   (or accepts them as arguments)
6. Runs `flask init-db` to create the database with the full schema

**`update.sh`** (deploy new version):
1. Backs up the current database (`backup.sh`)
2. Snapshots the current `app/` and `migrations/` to a `previous/` directory
   (overwriting any existing snapshot — only one rollback level is kept)
3. Copies updated `app/` and `migrations/` to the target (never touches
   `instance/` or `.env`)
4. Reinstalls Python dependencies (in case they changed)
5. Runs any pending migration scripts (see below)
6. Restarts the app process (see Process Model below)

**`rollback.sh`** (revert to previous deploy):
1. Checks that `previous/` exists; exits with error if no prior deploy to
   roll back to
2. Backs up the current database (`backup.sh`)
3. Restores `app/` and `migrations/` from `previous/`
4. Reinstalls Python dependencies to match the restored code
5. Restarts the app process
6. Note: database migrations are **not** reversed automatically. If a rollback
   requires undoing a migration, that must be handled manually with a
   corrective SQL script

**`backup.sh`**:
1. Copies `instance/app.db` to `instance/backups/app-<timestamp>.db`
2. WAL mode makes this safe while the app is running
3. Retains the last N backups (configurable, default 10)

### Database Migrations

Since there is no ORM migration tool, schema changes are managed with numbered
SQL scripts in `migrations/`:

```
migrations/
|-- 001_initial_schema.sql    # (identical to schema.sql, for reference)
|-- 002_add_event_description.sql
|-- 003_add_artist_genre.sql
`-- ...
```

A `schema_version` table tracks which migrations have been applied:

```sql
CREATE TABLE IF NOT EXISTS schema_version (
    version     INTEGER PRIMARY KEY,
    applied_at  TEXT NOT NULL DEFAULT (datetime('now')),
    filename    TEXT NOT NULL
);
```

The migration runner (a Flask CLI command `flask migrate` and also called by
`update.sh`):
1. Reads the current max `version` from `schema_version`
2. Finds all migration files with a number greater than the current version
3. Applies them in order within a transaction
4. Records each in `schema_version`

Initial install uses `schema.sql` directly. Migrations only apply to databases
that already exist.

### Process Model

- **Production**: Shared hosts typically provide Passenger (Phusion Passenger)
  or a similar WSGI runner that auto-manages the process. The app provides a
  `passenger_wsgi.py` entry point at the deploy root:

  ```python
  # passenger_wsgi.py
  from app import create_app
  application = create_app()
  ```

  If the host supports Gunicorn instead, a `run.sh` wrapper can start it on
  an assigned port. The scripts detect which method is available.

- **Development**: `flask run --debug` with auto-reload.

### Configuration

Environment variables loaded via python-dotenv from `.env`:

```bash
# .env.example
FLASK_SECRET_KEY=change-me-to-a-random-string
DATABASE=instance/app.db
FLASK_ENV=production
MAIL_SERVER=smtp.example.com
MAIL_PORT=587
MAIL_USE_TLS=true
MAIL_USERNAME=you@example.com
MAIL_PASSWORD=your-smtp-password
MAIL_DEFAULT_SENDER=noreply@example.com
```

`FLASK_SECRET_KEY` must be set. The app refuses to start without it in
production mode.

### SQLite in Production

- WAL mode is set on every connection (in `get_db()`).
- The database file lives in `instance/` (persisted across updates, never
  overwritten by the installer).
- If write contention becomes a problem (unlikely for this use case), the fix is
  to add a write lock retry with `PRAGMA busy_timeout=5000`.

### Shared Hosting Considerations

- **No root/sudo access.** Everything runs under the user account.
- **HTTPS** is typically handled by the hosting provider (e.g., Let's Encrypt
  via the host's control panel). The app does not manage TLS itself.
- **Process restarts** are handled by touching a `tmp/restart.txt` file
  (Passenger convention) or by the host's equivalent mechanism. The update
  and rollback scripts handle this automatically.
- **Static files** may be served directly by the host's web server (Apache/
  nginx) via an alias or `.htaccess` rule pointing to `app/static/`, avoiding
  Flask for static asset requests.

---

## Implementation Roadmap

Build in vertical slices. Each phase produces a working (if incomplete) app.

### Phase 1: Foundation

- Project scaffolding: `pyproject.toml`, `create_app` factory, `db.py`,
  `schema.sql`, `conftest.py`
- `flask init-db` CLI command
- Base template with Pico CSS and HTMX
- Auth blueprint: register, login, logout
- Email verification on registration (signed token via `itsdangerous`,
  sent via `smtplib`). This also establishes the email-sending
  infrastructure reused by password reset in Phase 5.
- User repository with password hashing
- Tests for auth flow (including verification happy path and expired token)

**Milestone: A user can register, verify their email, log in, and see an empty home page.**

### Phase 2: Core Data

- Venue, Artist, Show Run blueprints and repositories (full CRUD)
- Performance blueprint and repository (CRUD + link to artist/venue/show run)
- Event blueprint and repository (CRUD + M2M with performances)
- Templates for list, detail, and form views for each entity
- Tests for all CRUD operations and edge cases

**Milestone: A user can create venues, artists, and events with performances.**

### Phase 3: Personal Tracking

- Attendance blueprint/repository (plan, attend, cancel)
- Notes blueprint/repository (create, edit, delete with privacy levels)
- HTMX attendance toggle on performance pages
- HTMX inline note editing
- History/filter view: list attendance by date range, status, venue, artist

**Milestone: A user can track attendance and add notes to events.**

### Phase 4: Setlist FM Integration

- Setlist FM client service
- Autocomplete on artist, venue, and tour forms via Setlist FM search
- Setlist search and linking on performance pages
- Attribution display on pages showing Setlist FM-sourced data
- FTS5 search across artists, venues, events

**Milestone: Forms offer Setlist FM suggestions, performances can link setlists.**

### Phase 5: Polish

- Pagination on all list views (HTMX infinite scroll)
- Flash messages for all user actions
- Error pages (404, 500)
- Input validation tightening
- Performance: add `busy_timeout` pragma, review query plans for slow pages
- Password reset flow (email-based token, reset form, expiry)
- Security review: check all auth gates, CSRF coverage, input sanitization

**Milestone: App is ready for daily use by individual users.**

### Phase 6: Social Features

- Relationships repository (follow, unfollow, block)
- User profile page showing public attendance and notes
- Friends list page
- Note privacy enforcement (friends-only visibility)
- Block enforcement (hide data from blocked users)

**Milestone: Users can follow each other and see friends' public activity.**

### Phase 7: Calendar

- FullCalendar view with user's attendance data
- Friend overlay toggles on calendar
- .ics export endpoint with per-user token

**Milestone: Full calendar view and iCal export working.**

---

## Open Questions

These are decisions deferred until implementation makes them concrete:

1. **Image uploads**: No image support in v1. If added later, store in
   `instance/uploads/` and serve via the reverse proxy.

5. **App name**: The project still needs a real name. The repo is
   `setlist_collection_manager` from before the pivot. Rename when settled.
