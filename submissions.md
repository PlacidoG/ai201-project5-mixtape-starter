# Mixtape — Codebase Map

## Overview
Mixtape is a Flask + SQLAlchemy social music API: friends share songs, build
collaborative playlists, rate songs, and track listening streaks. It's a pure
JSON API (no templates/frontend) organized as routes (HTTP layer) calling
services (business logic layer) operating on SQLAlchemy models.

## Main files

| File | Responsibility |
|---|---|
| `app.py` | Application factory (`create_app`). Configures the DB URI, initializes `SQLAlchemy`, registers all four blueprints, and calls `db.create_all()`. This is the single source of the `db` object every other module imports. |
| `models.py` | All SQLAlchemy models: `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables (`friendships`, `song_tags`, `playlist_entries`). Every model has a `to_dict()` used to serialize API responses. |
| `routes/songs.py` | `GET /songs/search`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen`. Thin handlers that parse the request and delegate to `search_service` / `notification_service` / `streak_service`. |
| `routes/playlists.py` | `POST /playlists/`, `GET /playlists/<id>`, `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs`. Delegates to `playlist_service` and `notification_service.add_to_playlist`. |
| `routes/users.py` | `GET /users/<id>`, `GET /users/<id>/streak`, `GET /users/<id>/notifications`, `POST /users/notifications/<id>/read`. Delegates to `streak_service` and `notification_service`. |
| `routes/feed.py` | `GET /feed/<id>/listening-now`, `GET /feed/<id>/activity`. Delegates to `feed_service`. |
| `services/search_service.py` | Searches songs by title/artist substring match, joined with tags. |
| `services/streak_service.py` | Records listening events and maintains each user's consecutive-day listening streak. |
| `services/feed_service.py` | Computes the "friends listening now" (last 24h, deduped per friend) and general "activity" feeds by querying `ListeningEvent` filtered to the current user's friends. |
| `services/notification_service.py` | Creates/reads `Notification` rows; also owns `add_to_playlist` (adds a song to a playlist *and* notifies the sharer) and `rate_song`. |
| `services/playlist_service.py` | Creates playlists and returns their songs in position order. |
| `seed_data.py` | Wipes and repopulates the DB with 5 users, friendships, 25 songs (with varying tag counts), 3 playlists, listening events, and one seeded notification — run manually via `python seed_data.py`. |
| `tests/` | `pytest` suites (`test_streaks.py`, `test_search.py`, `test_playlists.py`) using an in-memory SQLite DB per test via a `create_app({"TESTING": True, "SQLALCHEMY_DATABASE_URI": "sqlite:///:memory:"})` fixture. |

## Data flow: adding a shared song to a playlist notifies its sharer

Note: this app has no "share a song" API endpoint — `Song` rows (and their
`shared_by` owner) only ever get created by `seed_data.py`. The real,
observable notification flow involving a shared song is triggered when
*another* user adds that song to a playlist:

1. Client calls **`POST /playlists/<playlist_id>/songs`** with `{song_id, added_by}`.
2. `routes/playlists.py` `add_song()` validates the two required fields are present, then calls `add_to_playlist(playlist_id, song_id, added_by)`.
3. `services/notification_service.py` `add_to_playlist()`:
   - Looks up the `Song`, the adding `User`, and the `Playlist` — raises `ValueError` (→ 400 at the route) if any is missing.
   - If the song isn't already in the playlist, appends it (`playlist.songs.append(song)`) and commits — this writes a `playlist_entries` row.
   - Compares `song.shared_by` to the adder's id. If they differ (i.e., someone other than the original sharer added it), calls `create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", body=...)`.
4. `create_notification()` builds and commits a new `Notification` row addressed to the original sharer.
5. Later, the sharer calls **`GET /users/<their_id>/notifications`** → `routes/users.py` → `notification_service.get_notifications()`, which queries `Notification` by `user_id` (optionally `read=False`), orders by `created_at` desc, and returns the list as JSON — surfacing the "so-and-so added your song to a playlist" message.

For contrast: `rate_song()` (same file, used by `POST /songs/<id>/rate`) never calls `create_notification` — rating a song produces no notification to its sharer, unlike adding it to a playlist.

## Other patterns worth knowing

- **Application factory + extension singleton**: `db = SQLAlchemy()` is module-level in `app.py`; `create_app()` calls `db.init_app(app)`. Every other module does `from app import db` rather than constructing its own instance.
- **Blueprint-per-resource routing**: each `routes/*.py` defines one `Blueprint` registered in `create_app()` with a URL prefix (`/songs`, `/playlists`, `/users`, `/feed`).
- **Thin routes, fat services**: route handlers only parse/validate request shape and translate exceptions to HTTP status codes; all business logic and DB queries live in `services/`.
- **Uniform error convention**: services raise plain `ValueError` for domain errors ("not found", "invalid score"); routes catch `ValueError` and return `jsonify({"error": str(e)})` with 400 or 404.
- **UUID string primary keys**: every model's `id` is a `db.String(36)` defaulting to `generate_uuid()` (`models.py`), not an auto-increment integer.
- **Association tables for many-to-many**: `friendships` (symmetric, self-referential on `User`), `song_tags`, and `playlist_entries` (which also carries `position`/`added_by`/`added_at` — an association table with extra columns, not a plain `db.relationship(secondary=...)`).
- **`to_dict()` serializers on every model**: routes never hand-build response JSON; they call the model's own `to_dict()`.
- **Test fixtures over mocks**: tests spin up a real (in-memory) SQLite DB per test via a `pytest` fixture rather than mocking the ORM.
