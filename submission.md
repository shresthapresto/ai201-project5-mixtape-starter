# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude throughout this project, mainly for codebase navigation and for explaining specific Python/SQLAlchemy behavior once I had already narrowed down a suspicious function.

- **Orientation**: I shared `models.py`, `app.py`, and the route files and asked for a summary of what each file/function was responsible for, and to trace the call chain for a couple of features (e.g. rating a song, adding a song to a playlist). This helped me build a mental model of the routes → services → models pattern before touching any issue.
- **Investigation**: For each bug, I read the relevant service file myself first and formed a hypothesis about where the problem was. I then asked Claude to explain specific pieces of behavior I wasn't 100% sure about — e.g. what `datetime.weekday()` returns for each day of the week, what a SQLAlchemy `.outerjoin()` does to row counts when the joined table has multiple matching rows, and why a list slice like `[:-1]` would behave the way it did.
- **Verification**: I did not accept any diagnosis without confirming it against the actual code and by reproducing/re-testing behavior myself, using `flask shell` to call service functions directly with controlled inputs and inspect real seeded data.
- **Where I double-checked**: For Issue #3 (duplicate search results), I made sure to actually check how many tags the duplicated song had versus the non-duplicated ones, rather than just accepting "it's the join" as an explanation — this confirmed the duplication count matched the tag count, which is what convinced me the root cause was correct.

---

## Codebase Map

**`app.py`** — Flask application factory. Initializes SQLAlchemy (`db`), loads config (DB URI, secret key), registers the four blueprints (`songs_bp`, `playlists_bp`, `users_bp`, `feed_bp`) under their URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and creates all tables on startup.

**`models.py`** — Defines all SQLAlchemy models: `User`, `Song`, `Tag`, `Playlist`, `Rating`, `ListeningEvent`, `Notification`, plus three association tables:
- `friendships` — symmetric many-to-many between users.
- `song_tags` — many-to-many between songs and tags.
- `playlist_entries` — many-to-many between playlists and songs, but with extra columns (`position`, `added_by`, `added_at`) — songs in a playlist have an explicit order, not just insertion order.

**`routes/`** — Thin HTTP layer. Each route file (`songs.py`, `playlists.py`, `users.py`, `feed.py`) parses the request, calls straight into a service function, and formats the JSON response. No business logic lives here — every route is a thin wrapper around one service call.

**`services/`** — All business logic lives here:
- `streak_service.py` — computes and updates listening streaks based on consecutive calendar days.
- `feed_service.py` — builds the "Friends Listening Now" feed and the general activity feed.
- `search_service.py` — searches songs by title/artist.
- `notification_service.py` — creates notifications (for playlist adds and ratings) and handles rating creation/update.
- `playlist_service.py` — creates playlists and retrieves ordered playlist contents.

**`seed_data.py`** — Populates the DB with 5 users, friendships, 25 songs (with varying tag counts), 3 playlists, and a mix of recent/old listening events — this is what makes several of the bugs reproducible (e.g. songs with 3+ tags exist specifically to expose Issue #3).

### Data flow — a user rates a song

1. `POST /songs/<song_id>/rate` hits `rate()` in `routes/songs.py`.
2. The route parses `user_id` and `score` from the JSON body and calls `rate_song(user_id, song_id, score)` in `services/notification_service.py`.
3. `rate_song()` validates the score (1–5), looks up the `Song` and `User`, checks for an existing `Rating` from that user for that song (update if found, otherwise create new), commits, and returns the `Rating`.
4. Originally, `rate_song()` stopped there — no notification was created. As part of Issue #4, I added a call to `create_notification()` before the commit, so the song's original sharer gets a `song_rated` notification, matching the pattern already used in `add_to_playlist()`.

### Pattern noticed

Every route file follows the same shape: parse input → call one service function → catch `ValueError` → return JSON with an appropriate status code. All actual logic (validation, DB queries, business rules) lives in `services/`, never in the routes. This made bug-hunting straightforward once I understood the pattern — nearly every bug in this project lived in a `services/*.py` file, exactly as the README stated.

---

## Root Cause Analysis

### Issue #1 — My listening streak keeps resetting

**How I reproduced it**: In `flask shell`, I set a user's `last_listened_at` to exactly 1 day before "now" (simulating "listened yesterday"), then called `record_listening_event()` for today. When "today" fell on a Sunday, the streak reset to 1 instead of incrementing, even though only one day had passed.

**How I found the root cause**: I traced `POST /songs/<id>/listen` → `record_listening_event()` → `update_listening_streak()` in `services/streak_service.py`. Reading the day-gap branch line by line, I found:
```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```
The `days_since_last == 1` check alone is a correct test for "listened on consecutive days." The extra `and today.weekday() != 6` condition doesn't belong in a consecutive-day check at all — I confirmed `datetime.weekday()` returns `6` for Sunday, so on Sundays this whole `elif` evaluates to `False` regardless of `days_since_last`, falling through to the `else` and resetting the streak.

**The root cause**: The streak-increment condition included an unrelated check, `today.weekday() != 6`, that excludes Sundays specifically. Any user who listened on consecutive days where the second day was a Sunday had their streak reset to 1 instead of incremented, because the day-of-week check overrode the correct consecutive-day logic.

**My fix and side-effect check**: I removed the `and today.weekday() != 6` clause, leaving `elif days_since_last == 1: user.listening_streak += 1`. I verified the `days_since_last == 0` (same-day, no change) and `days_since_last > 1` (gap, reset to 1) branches were untouched and still behave correctly, and re-ran `pytest tests/test_streaks.py`, which passed.

---

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it**: In `flask shell`, I created a `ListeningEvent` for a friend timestamped at 11pm the previous day (still within a rolling 24-hour window of "9am the next morning"), then called `get_friends_listening_now()`. The friend appeared in the results despite not having listened at all "today."

**How I found the root cause**: I opened `services/feed_service.py` and found:
```python
RECENT_THRESHOLD = timedelta(hours=24)
cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD
```
This computes a rolling 24-hour lookback from the current moment, not a "since midnight today" boundary. An event from 11pm yesterday is well within 24 hours of 9am the next day, so it passes the `listened_at >= cutoff` filter even though it happened on a different calendar day.

**The root cause**: The feed used a rolling 24-hour time window instead of a calendar-day boundary. Because the cutoff moved with the current time instead of resetting at midnight, events from late the previous evening remained "recent" well into the next morning.

**My fix and side-effect check**: I changed `cutoff` to be the start of the current UTC day (`datetime.now(timezone.utc).replace(hour=0, minute=0, second=0, microsecond=0)`) instead of `now - 24h`, and removed the now-unused `RECENT_THRESHOLD` constant. I checked `get_activity_feed()` in the same file — it doesn't use `cutoff` or `RECENT_THRESHOLD` at all (it just takes the most recent N events unfiltered by time), so it was unaffected. I re-tested with events just before and just after midnight to confirm the boundary now behaves correctly.

---

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it**: I called `GET /songs/search?q=Anthem` against the seeded data. "Crown Heights Anthem," which has 3 tags in `seed_data.py`, came back 3 times in the results, while songs with 0–1 tags came back once, matching the reported behavior exactly.

**How I found the root cause**: I opened `services/search_service.py` and found the query joins `Song` to `song_tags`:
```python
db.session.query(Song)
.outerjoin(song_tags, Song.id == song_tags.c.song_id)
.filter(...)
.all()
```
I confirmed with Claude's help that an `outerjoin` against a many-to-many association table produces one result row per matching row in the joined table — so a song with 3 tag associations produces 3 joined rows, and `.all()` returns 3 `Song` objects (one per row) instead of 1. I verified this by checking that the duplicate count for each song exactly matched its tag count in the seed data.

**The root cause**: The search query joined `Song` to the `song_tags` association table but never deduplicated the result set. Because the join fans out one row per tag, songs with N tags produced N identical rows in the results, while songs with 0 or 1 tags were unaffected (0 or 1 join rows), making the bug look "inconsistent" when it was actually determined entirely by tag count.

**My fix and side-effect check**: I added `.distinct()` to the query chain. I re-ran the search for "Anthem" and confirmed each song now appears exactly once regardless of tag count, and re-ran `pytest tests/test_search.py`, which passed.

---

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it**: I called `GET /playlists/<id>/songs` on a seeded playlist with 7 songs and got back only 6 — missing the one with the highest `position`. I then added a new song via `POST /playlists/<id>/songs` and re-fetched: the previously-missing song appeared, and the newly added song was now the one missing, matching the reported behavior.

**How I found the root cause**: I traced `GET /playlists/<id>/songs` → `get_playlist_songs()` in `services/playlist_service.py`. The query itself was correct — songs joined against `playlist_entries` and ordered by `position` ascending. The problem was the very last line:
```python
return [song.to_dict() for song in songs[:-1]]
```

**The root cause**: After correctly querying and ordering all songs in the playlist, the function sliced off the last element of the list (`songs[:-1]`) before returning it. This unconditionally dropped whichever song currently had the highest `position` — i.e., the most recently added song — regardless of how many songs were in the playlist.

**My fix and side-effect check**: I removed the `[:-1]` slice so the function returns the full ordered list: `[song.to_dict() for song in songs]`. I confirmed a playlist with 7 songs now returns all 7 in the correct order, and re-tested adding another song to confirm it appears immediately rather than being hidden. I re-ran `pytest tests/test_playlists.py`, which passed, and checked `create_playlist()` and `get_playlist()` in the same file, which don't touch this code path and were unaffected.

---

### Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it**: I called `POST /songs/<song_id>/rate` as a friend of the song's sharer, then checked `GET /users/<sharer_id>/notifications`. The rating was saved successfully (visible via `GET /songs/<song_id>`), but no new notification appeared, matching the reported behavior. As a control, I repeated the same flow for adding a song to a playlist, which did produce a notification.

**How I found the root cause**: I compared `add_to_playlist()` and `rate_song()` side by side in `services/notification_service.py`. `add_to_playlist()` performs its main action (appending the song to the playlist) and then explicitly calls `create_notification(...)` for the song's sharer. `rate_song()` performs its main action (creating/updating the `Rating`) and commits — but never calls `create_notification()` at all. This wasn't a typo or an off-by-one; the function simply never included the notification step that its sibling function had.

**The root cause**: `rate_song()` was missing a call to `create_notification()` entirely. The notification-creation pattern used for playlist adds was never implemented for ratings, so rating a song had no path that could ever produce a notification, regardless of who rated what.

**My fix and side-effect check**: I added a call to `create_notification()` inside `rate_song()`, placed after the rating is created/updated and before the commit, guarded by `if song.shared_by != user_id` (mirroring the same self-notification guard used in `add_to_playlist()`) so users don't get notified for rating their own songs. I verified a friend rating another user's song now produces a `song_rated` notification, and that a user rating their own shared song does not produce one. `get_notifications()` and `mark_as_read()` in the same file don't interact with this code path and were unaffected.