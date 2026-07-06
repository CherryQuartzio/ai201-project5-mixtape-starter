# Mixtape Bug Hunt - Submission

## AI Usage

AI agent is primarily used in this project to help trace execution flows and to verify existing bugs.

**Example 1**
- What I asked: To walk me through the execution flow for a use case, in which there could be bugs presence
- What it did: Generated a brief explanation of which part in the code in what order the program executes in sequence. This helps to harden my understanding of the codebase.
- Verification: Going through the code by myself in some occasions to check for correctness of the AI responses.

**Example 2**
- What I asked: To verify whether a certain bug I think I found is an actual bug
- What it did: Perform test API run to try to replicate the entrypoint for the bug
- Verification: Ran the exact test the AI ran to confirm the finding

## Codebase Map

### Main files and their responsibilities

- **`app.py`** - Flask application factory (`create_app`). Configures the SQLite DB URI, initializes the `db` SQLAlchemy instance, registers the four route blueprints (`songs`, `playlists`, `users`, `feed`) under their URL prefixes, and calls `db.create_all()`. This is why the app must be started with `flask run` (via `FLASK_APP=app:create_app`) rather than `python app.py` - running the module directly re-imports it under `__main__` and creates a second, conflicting `SQLAlchemy` registration.

- **`models.py`** - Defines all 7 SQLAlchemy models/tables:
  - `User` - has `listening_streak` and `last_listened_at` (used by the streak feature), plus a self-referential many-to-many `friends` relationship via the `friendships` association table.
  - `Song` - has a many-to-many `tags` relationship via the `song_tags` association table, and a `shared_by` FK to the `User` who posted it.
  - `Tag` - simple lookup table for genres/moods.
  - `ListeningEvent` - one row per "user listened to song at time T"; this is what both the streak service and the feed service read from.
  - `Rating` - one row per (user, song) rating, enforced unique via `UniqueConstraint("user_id", "song_id")`.
  - `Playlist` - has a many-to-many `songs` relationship via the `playlist_entries` association table, which (unlike the other two association tables) carries extra columns: `position` (ordering), `added_by`, and `added_at`.
  - `Notification` - one row per notification, with a `notification_type` string and a `read` boolean.
  Every model has a `to_dict()` used to serialize it for JSON responses.

- **`routes/`** - Thin HTTP layer. Each route function pulls parameters out of `request`, calls exactly one service function, and translates the result (or a caught `ValueError`) into a JSON response with the right status code. No business logic lives here.
  - `songs.py` - `/songs/search`, `/songs/<id>`, `/songs/<id>/rate`, `/songs/<id>/listen`
  - `playlists.py` - `/playlists/` (create), `/playlists/<id>`, `/playlists/<id>/songs` (get/post)
  - `users.py` - `/users/<id>`, `/users/<id>/streak`, `/users/<id>/notifications`, `/users/notifications/<id>/read`
  - `feed.py` - `/feed/<id>/listening-now`, `/feed/<id>/activity`

- **`services/`** - All actual business logic lives here. This is where the 5 tracked bugs live.
  - `streak_service.py` - `record_listening_event()` creates a `ListeningEvent` and calls `update_listening_streak()`, which compares `last_listened_at` to "today" to decide whether to increment, reset, or leave the streak unchanged.
  - `feed_service.py` - `get_friends_listening_now()` (24h-recency-filtered, deduped to one entry per friend) and `get_activity_feed()` (last N events regardless of recency).
  - `search_service.py` - `search_songs()` (title/artist `ILIKE` match, outer-joined against `song_tags`) and `get_song()`.
  - `notification_service.py` - `create_notification()` is the shared primitive; `add_to_playlist()` and `rate_song()` are the two "trigger" functions that are supposed to call it when a friend interacts with your shared song.
  - `playlist_service.py` - `create_playlist()`, `get_playlist_songs()` (returns songs in `position` order), `get_playlist()`, `get_user_playlists()`.

- **`seed_data.py`** - Populates 5 friended users, 25 songs (deliberately split into 0-tag / 1-tag / 3+-tag groups - the comments call out that the 3+-tag group is what "exposes Issue #3"), 3 playlists of 5–7 songs each, a mix of recent (within 30 min) and older (1–14 days) listening events, pre-set streaks, and one working "song added to playlist" notification "so students can see the correct pattern when investigating Issue #4."

- **`tests/`** - `test_streaks.py`, `test_search.py`, `test_playlists.py`. Existing pytest coverage to run before/after fixes.

### Pattern observed across the codebase

Every service function that looks up a row by ID follows the same shape: fetch with `db.session.get(...)`, raise `ValueError(f"... not found")` if `None`, otherwise proceed. Every route function wraps its service call in `try/except ValueError` and turns that into a `400`/`404` JSON error. This means the routes layer is almost entirely mechanical - to understand *why* an endpoint behaves a certain way, the actual logic is always one level down in `services/`, never in `routes/`.

### Data flow trace #1: a user rates a song

1. Client sends `POST /songs/<song_id>/rate` with JSON body `{user_id, score}`.
2. `routes/songs.py:rate()` pulls `user_id` and `score` out of the request body, validates both are present, then calls `notification_service.rate_song(user_id, song_id, int(score))`.
3. `notification_service.rate_song()` validates `1 <= score <= 5`, loads the `Song` and the rating `User`, checks for an existing `Rating` row for that `(user_id, song_id)` pair (upserts if found, inserts if not), commits, and returns the `Rating`.
4. The route serializes the returned `Rating` via `.to_dict()` and responds `201`.

Notably, `rate_song` lives in `notification_service.py` (not a "ratings service") because the intent is that rating a song should also notify whoever originally shared it - that's the pattern `add_to_playlist()` in the same file follows (it calls `create_notification()` after adding the song). `rate_song()` currently has no such call. This is the comparison the Issue #4 hint points at.

### Data flow trace #2: a user views a playlist's songs

1. Client sends `GET /playlists/<playlist_id>/songs`.
2. `routes/playlists.py:get_songs()` calls `playlist_service.get_playlist_songs(playlist_id)`.
3. `get_playlist_songs()` loads the `Playlist` (404s if missing), then queries `Song` joined against the `playlist_entries` association table, filtered to this playlist and ordered ascending by `position`.
4. The route wraps the returned list in `{"songs": [...], "count": N}` and responds `200`.

## Issue Triage (from initial read-through, before reproduction)

| # | Title | Affected file | Note from orientation |
|---|-------|---------------|------------------------|
| 1 | Listening streak keeps resetting | `streak_service.py` | `update_listening_streak()` has a `today.weekday() != 6` special case guarding the increment branch - worth checking against Python's weekday semantics. |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` | Uses a 24h rolling cutoff (`RECENT_THRESHOLD`), not a calendar-day boundary - seed data has both very recent (<30 min) and older (1–14 day) events to test against. |
| 3 | Same song shows up twice in search | `search_service.py` | `search_songs()` does an `outerjoin` against `song_tags` before `.all()` - seed data explicitly includes songs with 3+ tags "to expose Issue #3," suggesting a fan-out from the join. |
| 4 | No notification on rating (but works for playlist-add) | `notification_service.py` | `add_to_playlist()` calls `create_notification()`; `rate_song()` does not. Structural, not a typo, per the hint. |
| 5 | Last song in a playlist never shows up | `playlist_service.py` | `get_playlist_songs()` returns `songs[:-1]` - an explicit slice dropping the final element. |

These are hypotheses only - Milestone 2 will confirm each by actually reproducing the behavior before any fix is attempted.

## Milestone 2: Reproduction

Before writing any fix, I reproduced each bug - first the original primary three, then adjusted the plan after Issue #3 refused to reproduce.

### Issue #3 (duplicate search) - could NOT reproduce in this environment

Attempted reproduction: called `search_songs("Crown Heights")` against a song with 3 tags (seed data + `tests/test_search.py`'s fixture both set this up specifically to trigger the bug - the test file even comments `# Should be 1, bug causes it to be 3`).

- Confirmed the `outerjoin` against `song_tags` does fan out at the raw SQL/cursor level: querying the compiled statement directly via `db.session.execute(q.statement)` returns **3** identical rows for a 3-tagged song.
- But calling `search_songs()` itself (which goes through the legacy `Query.all()` API) returns exactly **1** result for the same song - no duplication.
- Ran the existing `tests/test_search.py` suite: **all 5 tests pass**, including `test_search_no_duplicates_multi_tag_song`, which is written to fail if the bug is present.

Root cause of the non-reproduction: this environment has `sqlalchemy==2.0.51`. Modern SQLAlchemy's legacy `Query.all()` automatically de-duplicates ORM entity results by primary key when a plain join fans out a single-entity query - a behavior that didn't exist in older SQLAlchemy versions the bug was presumably written against. The `outerjoin` in `search_songs()` is still dead/pointless code (it filters on `Song.title`/`Song.artist` only, never on `song_tags`), but it's no longer *user-visible* as a duplicate-results bug here.

Per the Milestone 2 guidance ("try a different one from the list if you can't reproduce a bug after a genuine attempt"), I substituted **Issue #1 (streak)** as the third primary bug instead.

### Issue #1 (listening streak keeps resetting) - reproduced

Isolated `update_listening_streak()` directly (per the "Isolate" strategy in the hints) with controlled inputs instead of depending on wall-clock timing:

```python
user.listening_streak = 5
user.last_listened_at = datetime(2026, 7, 4, 20, 0, tzinfo=timezone.utc)  # a Saturday
update_listening_streak(user, datetime(2026, 7, 5, 20, 0, tzinfo=timezone.utc))  # the next day, a Sunday
```

Expected: streak increments to 6 (consecutive day). Actual: streak resets to **1**.

Also confirmed via the existing suite: `pytest tests/test_streaks.py` - `test_streak_increments_on_sunday` **fails** (`assert 1 == 2`), reproducing the exact same behavior independently.

### Issue #4 (no notification on rating) - reproduced

Started the app (`flask run`) and drove it over HTTP:

1. `GET /users/<simone_id>/notifications` → `{"count": 0, "notifications": []}`
2. `POST /songs/<crown_heights_id>/rate` as `nova` (a friend, not the sharer) with `{"user_id": nova_id, "score": 5}` → `201`, rating created successfully.
3. `GET /users/<simone_id>/notifications` again (simone is the song's sharer) → still `{"count": 0, "notifications": []}`.

Confirmed as a genuine gap: `simone` never gets notified even though the rating succeeded. (No existing automated test covers notifications - there's no `test_notifications.py`.)

### Issue #5 (last song in playlist never shows up) - reproduced

Queried the DB directly to get ground truth, then hit the endpoint:

- `playlist_entries` table has **7** rows for playlist "Late Night Vibes" (`001d7eb2-...`).
- `GET /playlists/001d7eb2-.../songs` → `{"count": 6, ...}` - one song short, and it's specifically the entry with the highest `position` (last-added) that's missing from the response.

Also confirmed via the existing suite: `pytest tests/test_playlists.py` - `test_playlist_returns_all_songs` fails (`4 == 5`) and `test_playlist_returns_songs_in_order` fails (missing `"Track 5"`), both matching the same off-by-one pattern.

### Full test suite baseline (before any fix)

```
tests/test_playlists.py::test_playlist_returns_all_songs        FAILED  (4 == 5)
tests/test_playlists.py::test_playlist_returns_songs_in_order    FAILED  (missing "Track 5")
tests/test_streaks.py::test_streak_increments_on_sunday          FAILED  (1 == 2)
3 failed, 10 passed
```

## Bug selection (final, after reproduction)

Primary three: **#1 (streak)**, **#4 (missing rating notification)**, **#5 (last song missing)** - all three reproduced cleanly and deterministically.
Not pursuing: **#3** (does not reproduce under the installed SQLAlchemy version - see above). **#2 (feed recency)** remains a stretch candidate; a quick manual check of `get_friends_listening_now()` against seed data didn't surface an obvious discrepancy, so it needs more investigation time than #1/#4/#5 before I'd commit to it.

## Root Cause Analysis Entries

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:** Queried the `playlist_entries` table directly for playlist "Late Night Vibes" (`001d7eb2-...`) and found 7 rows. Called `GET /playlists/001d7eb2-.../songs` and got `count: 6` back - one song short, specifically the entry with the highest `position` value. Also ran the existing `tests/test_playlists.py` suite and found `test_playlist_returns_all_songs` (`4 == 5`) and `test_playlist_returns_songs_in_order` (missing `"Track 5"`) both failing with the same pattern.

**How I found the root cause:** Traced `GET /playlists/<id>/songs` → `routes/playlists.py:get_songs()` → `playlist_service.get_playlist_songs()`. The function's own docstring says "This function returns all songs in the playlist," which immediately contradicted the observed missing-last-song behavior, so I read the function body line by line looking for anything that would trim the result after the ordered query ran.

**The root cause:** After querying songs ordered ascending by `position`, the function returned `songs[:-1]` instead of `songs` (`services/playlist_service.py`, `get_playlist_songs`) - an explicit slice that unconditionally discards the last element of the list, regardless of playlist size. Since the query already orders ascending by position, the last element of the list is always the most-recently-added song (highest position), so that song was silently dropped from every playlist response.

**Fix and side-effect check:** Changed `return [song.to_dict() for song in songs[:-1]]` to `return [song.to_dict() for song in songs]`. Verified: `pytest tests/test_playlists.py` now shows 3/3 passing (previously 1/3). Checked the boundary this bug most directly threatened - a playlist with exactly 1 song - by seeding one in an isolated in-memory DB and confirming `get_playlist_songs()` now returns that 1 song instead of the empty list the old `[:-1]` slice would have produced. Re-ran the full `pytest tests/` suite: 12/13 passing (the only remaining failure is the not-yet-fixed streak test, unrelated to this change). Re-drove the original HTTP repro against the live server: `GET /playlists/001d7eb2-.../songs` now returns `count: 7` including "Free Throws," the previously-missing song.

### Issue #1: My listening streak keeps resetting

**How I reproduced it:** Isolated `update_listening_streak()` and called it directly with controlled inputs (per the "Isolate" strategy) instead of relying on wall-clock timing: gave a user a streak of 5 with `last_listened_at` set to a Saturday, then called the function with `now` set to the following Sunday. Expected the streak to increment to 6 (a consecutive day); actual result was a reset to 1. Independently confirmed via the existing `tests/test_streaks.py::test_streak_increments_on_sunday`, which failed with `assert 1 == 2` before any fix.

**How I found the root cause:** The function's docstring states the rule plainly - "If the user listened yesterday: streak increments by 1" - with no day-of-week exception. Reading the `elif` branch that handles the one-day-gap case, I found an extra condition, `and today.weekday() != 6`, that isn't implied by the docstring at all, so I checked what `.weekday()` returns for Sunday.

**The root cause:** Python's `date.weekday()` returns `6` for Sunday (Monday=0 ... Sunday=6). The increment branch in `update_listening_streak()` (`services/streak_service.py`) was gated by `days_since_last == 1 and today.weekday() != 6`, so whenever "today" happened to be a Sunday, this condition evaluated to `False` even though the user listened on a consecutive day - the code fell through to the `else` branch and reset the streak to 1 instead of incrementing it. There is no documented rule that Sundays should behave differently; the weekday check is simply an erroneous condition with no legitimate purpose.

**Fix and side-effect check:** Removed the `and today.weekday() != 6` clause, so the branch now reads `elif days_since_last == 1:`. Verified: `pytest tests/test_streaks.py` now shows 5/5 passing (previously 4/5). Explicitly checked both sides of the boundary this bug touched - Saturday→Sunday (the failing case, now correctly increments 5→6) and Sunday→Monday (already worked before the fix, confirmed it still does, now 3→4). Ran the full `pytest tests/` suite: 13/13 passing.

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** Drove the app over HTTP. Checked `GET /users/<simone_id>/notifications` (`count: 0`), then had a friend (`nova`, not the song's sharer) call `POST /songs/<song_id>/rate` on a song `simone` shared - got back a `201` with a valid `Rating` object, confirming the rating itself succeeded. Checked `GET /users/<simone_id>/notifications` again - still `count: 0`. No existing automated test covers this (there's no `test_notifications.py`), so this was purely a manual repro.

**How I found the root cause:** Per the hint, compared `rate_song()` line-by-line against `add_to_playlist()` in the same file (`services/notification_service.py`), since both represent "a friend interacted with your shared song" and `add_to_playlist()` is the one that already works. `add_to_playlist()` follows a clear pattern: mutate state, then call `create_notification(user_id=song.shared_by, ...)` guarded by `if song.shared_by != added_by_user_id` so the sharer doesn't get notified about their own action. `rate_song()` mutates state (creates/updates the `Rating`, commits) but the function simply ends there - there's no `create_notification()` call anywhere in it.

**The root cause:** This is architectural, not a typo, as the hint suggested - `rate_song()` was never given the notify step that every other "friend interacted with your song" action follows. The notification-creation call was omitted entirely from the function, not miswired or misconfigured.

**Fix and side-effect check:** Added, after the existing `db.session.commit()` in `rate_song()`:
```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```
This mirrors `add_to_playlist()`'s guard so a user rating their own shared song does not self-notify. Verified via HTTP: `darius` rating `simone`'s song took her notification count from 0 → 1 with a correct `song_rated` body; `simone` then rating her *own* song left the count unchanged (self-notify guard confirmed). Re-verified `add_to_playlist()`'s pre-existing notification path still fires correctly and is unaffected by this change (`kenji` re-adding an already-playlisted song still notified `darius`, the sharer). Ran the full `pytest tests/` suite: 13/13 passing (no regressions; no existing tests cover notifications either way).

**Aside (out of scope):** while verifying, `POST /playlists/<id>/songs` for a song *not already in the playlist* threw a `500` (`IntegrityError: NOT NULL constraint failed: playlist_entries.position`) - `add_to_playlist()`'s `playlist.songs.append(song)` doesn't populate the `position`/`added_by` columns that `playlist_entries` requires beyond the two FK columns. This is a real, pre-existing bug (confirmed unrelated to any of my changes), but it isn't one of the 5 tracked issues, so I left it as-is rather than fixing it in scope.
