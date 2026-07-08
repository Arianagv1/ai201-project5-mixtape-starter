# Mixtape — Codebase Map

Mixtape is a Flask + SQLAlchemy JSON API for a social music-sharing app. Users share
songs, add them to collaborative playlists, rate them, listen to them (building daily
streaks), and see what their friends are playing in a feed.

## Main files and their responsibilities

### `app.py` — application factory + DB handle
Defines the shared `db = SQLAlchemy()` instance and the `create_app(config=None)` factory.
The factory sets config (SQLite `mixtape.db` by default, overridable via `DATABASE_URL`),
calls `db.init_app(app)`, registers the four blueprints under URL prefixes
(`/songs`, `/playlists`, `/users`, `/feed`), and runs `db.create_all()`. Because `db`
lives here and the models import it, the app **must** be started through the factory
(`FLASK_APP=app:create_app flask run`) — importing `app.py` as a script double-registers
the models and triggers a SQLAlchemy error.

### `models.py` — the data layer
Seven entities plus three association tables, all keyed by string UUIDs:

- **`User`** — username, email, `listening_streak`, `last_listened_at`. Has relationships
  to shared songs, ratings, listening events, notifications, playlists, and a self-
  referential many-to-many `friends` (via the `friendships` table, stored bidirectionally).
- **`Song`** — title, artist, album, genre, `shared_by` (FK to the sharer), `share_note`.
  Related to ratings, listening events, and tags.
- **`Tag`** — a name; linked to songs via the `song_tags` join table.
- **`ListeningEvent`** — a `(user_id, song_id, listened_at)` row. This is the raw material
  the feed is computed from.
- **`Rating`** — a `(user_id, song_id, score 1–5)` row with a unique constraint on
  `(user_id, song_id)`, so a user has at most one rating per song. Ratings are their own
  model, **not** a column on `Song`.
- **`Playlist`** — name, `created_by`, `is_collaborative`. Songs are attached via the
  **`playlist_entries`** join table, which carries an explicit `position` column plus
  `added_by` and `added_at` — so playlist membership records ordering and provenance, not
  just insertion.
- **`Notification`** — `user_id` (recipient), `notification_type`, `body`, `read` flag.

Association tables: `friendships`, `song_tags`, and `playlist_entries` (the last is the
richest — it's an ordered, attributed join table).

### `routes/` — thin HTTP layer (blueprints)
Each blueprint parses input, calls one service function, and formats the JSON response.
Errors surface as `ValueError` from the service and are translated to 400/404.

- **`routes/songs.py`** — `GET /songs/search?q=`, `GET /songs/<id>`,
  `POST /songs/<id>/rate`, `POST /songs/<id>/listen`.
- **`routes/playlists.py`** — `POST /playlists/`, `GET /playlists/<id>`,
  `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs` (add a song).
- **`routes/users.py`** — `GET /users/<id>`, `GET /users/<id>/streak`,
  `GET /users/<id>/notifications`, `POST /users/notifications/<id>/read`.
- **`routes/feed.py`** — `GET /feed/<id>/listening-now`, `GET /feed/<id>/activity`.
- **`routes/__init__.py`** — empty package marker.

### `services/` — business logic
- **`feed_service.py`** — `get_friends_listening_now(user_id)` (each friend's single most
  recent song within the last 24h, deduped) and `get_activity_feed(user_id, limit=20)`
  (most recent N friend events, no recency cutoff, not deduped).
- **`streak_service.py`** — `record_listening_event()` (writes a `ListeningEvent` and
  updates the streak), `update_listening_streak()` (the +1 / reset rules), `get_streak()`.
- **`notification_service.py`** — `create_notification()`, `add_to_playlist()` (adds a song
  to a playlist and notifies the sharer), `rate_song()` (upsert a rating),
  `get_notifications()`, `mark_as_read()`.
- **`playlist_service.py`** — `create_playlist()`, `get_playlist_songs()` (ordered by
  `position`), `get_playlist()`, `get_user_playlists()`.
- **`search_service.py`** — `search_songs()` (case-insensitive title/artist match with
  tags) and `get_song()`.

### `seed_data.py` — test fixtures
Drops and recreates all tables, then seeds 5 users with friendships, 10 tags, songs with
0 / 1 / 3+ tags, recent and older listening events, streak state, three playlists, and a
sample notification. Run with `python seed_data.py`.

### `tests/`
`test_playlists.py`, `test_search.py`, `test_streaks.py` exercise the corresponding
service behavior.

## Data flow — adding a song to a playlist triggers a notification

This is the app's real notification path. Say **darius** adds one of **nova's** shared
songs to nova's playlist.

1. **`POST /playlists/<playlist_id>/songs`** with `{"song_id": ..., "added_by": <darius>}`
   hits `add_song()` in [routes/playlists.py](routes/playlists.py#L43). The route validates
   that `song_id` and `added_by` are present, then delegates.
2. **`notification_service.add_to_playlist(playlist_id, song_id, added_by)`**
   ([services/notification_service.py](services/notification_service.py#L35)):
   1. Loads the `Song`, the adder (`User`), and the `Playlist`, raising `ValueError` if any
      is missing.
   2. Appends the song to `playlist.songs` (if not already present) and commits — this
      inserts a `playlist_entries` row.
   3. Compares `song.shared_by` to `added_by`. If they differ, it calls
      **`create_notification(user_id=song.shared_by, type="song_added_to_playlist", body=...)`**
      — so the notification goes to the song's **original sharer** (nova), not the adder.
3. **`create_notification()`** inserts a `Notification` row and commits.
4. The route returns `{"message": "Song added to playlist"}` with `201`.
5. Later, nova calls **`GET /users/<nova>/notifications`** → `get_notifications()` reads the
   `Notification` rows back, newest first.

A second real flow — **listening builds the feed**: `POST /songs/<id>/listen` →
`streak_service.record_listening_event()` writes a `ListeningEvent` and bumps the streak;
a friend then sees that song via `GET /feed/<id>/listening-now` →
`feed_service.get_friends_listening_now()`, which reads recent `ListeningEvent` rows for the
viewer's friends. There is no stored "feed" table — the feed is computed on read.

## Patterns in how the app is organized

- **Strict layering: routes → services → models.** Every route immediately delegates to a
  service function. Routes only parse request input and format JSON responses; all business
  logic and DB access lives in `services/`. Routes never query models directly (the one
  small exception is `users.py`, which does a trivial `db.session.get(User, ...)` lookup).
- **`ValueError` as the cross-layer error contract.** Services raise `ValueError("… not
  found")` for missing entities; every route wraps its service call in `try/except
  ValueError` and maps it to a 404 (or 400 on writes). No custom exception types.
- **Notifications are a side effect, not a resource you create directly.** There's no
  "create notification" endpoint. They're produced internally by `add_to_playlist` when one
  user acts on another user's song.
- **Derived-on-read feeds.** The feed isn't materialized; it's queried live from
  `ListeningEvent`, with the two endpoints differing only in filtering (24h + dedup vs.
  most-recent-N).
- **UUID string PKs and `to_dict()` everywhere.** Every model generates a UUID primary key
  and exposes a `to_dict()` used for JSON serialization, keeping response shaping in the
  model rather than the route.
- **Application-factory + blueprint modularity.** `create_app()` plus one blueprint and one
  service module per domain (songs, playlists, users, feed) makes the feature boundaries
  explicit and testable.

📋 Root Cause Analysis Format

Issue number and title

How you reproduced it — What steps did you take to confirm the bug exists before touching any code? What inputs, sequence of actions, or data condition triggered the behavior?

How you found the root cause — Which files did you look at? What was your navigation path? What moment made you confident you'd found the right place — not just a suspicious area, but the specific cause?

The root cause — In plain English, explain exactly what was wrong. Not "there was a bug in the streak logic" — explain the specific condition, comparison, or missing step that caused the problem.

Your fix and side-effect check — What did you change and why does that change fix the root cause? What related functionality did you check afterward to confirm you didn't break anything?