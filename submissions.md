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

## Bugs found — Root Cause Analysis

I investigated all five issues from the tracker and **reproduced four of them** (Issues #1, #2, #4, #5). Issue #3 turned out **not to reproduce** through the actual code path but was a genuine latent defect, so it was hardened defensively. Reproduction was done two ways: (a) the checked-in test suite, where `python -m pytest tests/` fails on the streak and playlist bugs, and (b) a small in-memory harness (a throwaway script using the same `sqlite:///:memory:` app fixture) for the feed and notification bugs, which have no test file.

Baseline test run before any fix: **3 failed, 10 passed** — `test_streak_increments_on_sunday`, `test_playlist_returns_all_songs`, `test_playlist_returns_songs_in_order`. After all five fixes: **13 passed, 0 failed**.

Each entry below has all five required fields: how I reproduced it, how I found the root cause, the root cause, and my fix + side-effect check.

### Issue #1 — Listening streak resets every Sunday
- **How I reproduced it:** Ran `python -m pytest tests/test_streaks.py` before touching code. The checked-in test `test_streak_increments_on_sunday` calls `update_listening_streak` with Saturday `2024-06-15` then Sunday `2024-06-16` and asserts the streak is `2`; it failed with `assert 1 == 2`, confirming that listening on a Sunday after a Saturday resets the streak instead of incrementing it. Real-app equivalent: a user with `last_listened_at` on a Saturday who `POST /songs/<id>/listen`s the following Sunday, then reads `GET /users/<id>/streak`.
- **How I found the root cause:** Followed the flow from the failing test into `services/streak_service.py`. `record_listening_event` delegates the streak math to `update_listening_streak(user, now)`, so I read that function. The branch structure computes `days_since_last = (today - last_date).days` and then decides: `==0` no-op, `==1` increment, else reset. The moment of certainty was reading the increment guard, `elif days_since_last == 1 and today.weekday() != 6:` — the second clause has nothing to do with whether the previous day was consecutive, and `weekday() == 6` is exactly Sunday, matching the "only on Sundays" symptom precisely.
- **The root cause:** The consecutive-day increment branch carried an extra, unjustified condition `and today.weekday() != 6`. `date.weekday()` returns `6` for Sunday, so whenever the *current* listen falls on a Sunday, the "listened yesterday → increment" branch is skipped and control falls through to the `else`, which hard-resets `listening_streak = 1`. Every other weekday incremented correctly; only Sunday broke.
- **My fix and side-effect check:** Removed the spurious clause so the branch is simply `elif days_since_last == 1:` ([services/streak_service.py:73](services/streak_service.py#L73)). This restores the intended rule: any listen exactly one calendar day after the last one increments, regardless of weekday. Side-effect check — reran the full `tests/test_streaks.py` (5/5 pass), covering both sides of every boundary: same-day (`==0`) still no-ops (`test_streak_does_not_double_count_same_day`), consecutive day still increments (`test_streak_increments_on_consecutive_day` and now `test_streak_increments_on_sunday`), and a skipped day still resets (`test_streak_resets_after_skipped_day`). `last_listened_at` is still updated on every non-same-day branch, so `get_streak` and the `POST /songs/<id>/listen` flow are unaffected.

### Issue #2 — "Friends Listening Now" shows people from up to a day ago
- **How I reproduced it:** There's no test file for the feed, so I wrote a throwaway in-memory harness (same `sqlite:///:memory:` fixture). I created a user with two friends, gave one friend a single `ListeningEvent` at `now - 20h` (and no fresher event), and called `get_friends_listening_now(me_id)`. It returned **1** entry — the friend shown as "listening now" to a 20-hour-old track (expected `0`). Trigger condition: a friend whose *most recent* event lies between the intended live window and 24h ago; per-friend dedup means a fresher event would otherwise mask it.
- **How I found the root cause:** Traced `GET /feed/<id>/listening-now` → `routes/feed.py` → `feed_service.get_friends_listening_now`. Reading the query, the only recency gate is `.filter(ListeningEvent.listened_at >= cutoff)` where `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`. I jumped to the module constant and saw `RECENT_THRESHOLD = timedelta(hours=24)` — a full day, which is what let "yesterday" through. The seed file corroborated it: [seed_data.py:121](seed_data.py#L121) comments that older (2h–14d) events "should NOT appear in 'listening now' after fix," and recent ones are seeded within 30 minutes.
- **The root cause:** The "listening now" recency window was set to 24 hours. For a real-time feed that is far too wide — any friend who listened at any point in the last day (i.e. yesterday) qualifies as "listening now."
- **My fix and side-effect check:** Changed the constant to `RECENT_THRESHOLD = timedelta(minutes=30)` ([services/feed_service.py:13](services/feed_service.py#L13)), matching the seed data's notion of "recent" (≤30 min appears, ≥2h does not). Boundary check via the harness: a listen 10 min ago is included, a listen 20h ago is excluded. Side-effect check — `get_activity_feed` shares the module but does **not** use `RECENT_THRESHOLD` (it has no cutoff), and I confirmed it still returns both events (count `2`), so the historical activity feed is unaffected; the full test suite showed no new failures.

### Issue #3 — "Same song shows up twice in search" — investigated, does NOT reproduce
- **Location:** `services/search_service.py:25-35`
- **Symptom (as reported):** A song appears multiple times in search results, seemingly at random.
- **What I found:** The query does `db.session.query(Song).outerjoin(song_tags, ...)`, which fans out one row per tag — so the *raw* SQL genuinely returns 3 rows for a 3-tag song (I confirmed: `db.session.query(Song.id).outerjoin(song_tags…)` returns **3**). **However**, the reported user-visible bug does not actually occur: SQLAlchemy's legacy `db.session.query(Song)` interface deduplicates entity rows by identity before returning them, so `search_songs("Crown Heights")` returns the 3-tag song **once**. The checked-in test `tests/test_search.py::test_search_no_duplicates_multi_tag_song` (which expects `1`) **passes**.
- **How I attempted it:** Ran `search_songs("Crown Heights")` against a seeded 3-tag song → 1 result (not 3). Also ran the raw column query to confirm the underlying fan-out is real (3 rows) but masked by ORM entity uniquing.
- **Status:** Latent defect (the unnecessary un-`distinct()`ed join) but **not currently reproducible** through the service/API on this SQLAlchemy version. Per the "try a different one if you can't reproduce it" guidance, I documented the other four instead. Worth noting it would resurface if the query were migrated to the 2.0 `select()` style, where entity uniquing is not automatic unless `.unique()` is called.

### Issue #4 — No notification when a friend rates your shared song
- **Location:** `services/notification_service.py:73-110` (`rate_song`)
- **Symptom:** Adding someone's song to a playlist notifies the sharer, but rating their song does not.
- **Root cause:** `rate_song()` creates/updates the `Rating` and commits, but never calls `create_notification()`. The sibling `add_to_playlist()` (`services/notification_service.py:64-70`) *does* notify `song.shared_by`, so the rating path is simply missing the equivalent notification call.
- **How I reproduced it:** In the in-memory harness, a `sharer` shared a song; a different `rater` called `rate_song(rater_id, song_id, 5)`. `get_notifications(sharer_id)` returned length `0` both before and after — no notification was created.
- **Expected vs actual:** expected a `song_rated` notification for the sharer (count `0 → 1`); actual no change (`0 → 0`).

### Issue #5 — The last song in a playlist never shows up
- **Location:** `services/playlist_service.py:66`
- **Symptom:** A playlist's song list is always missing its final (most recently positioned) track.
- **Root cause:** `return [song.to_dict() for song in songs[:-1]]` — the `[:-1]` slice drops the last element of the position-ordered list. It should iterate all of `songs`.
- **How I reproduced it:** The checked-in tests `tests/test_playlists.py::test_playlist_returns_all_songs` (seeds 5 songs, asserts `len == 5`) and `test_playlist_returns_songs_in_order` both fail — the first returns 4, the second is missing `"Track 5"`. Real-app equivalent: `GET /playlists/<id>/songs` on any non-empty seeded playlist returns one fewer song than were added. Empty playlists are unaffected because `[][:-1] == []`.
- **Expected vs actual:** expected all N songs; actual N−1 (a 1-song playlist returns 0).
