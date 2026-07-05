# Project 5: Mixtape Bug Hunt — Submission

## AI Usage

I used Claude in two distinct ways during this project, and I want to be honest about where it helped and where I had to correct it.

**Codebase navigation and tracing.** During orientation and investigation, I used Claude to trace call chains and explain functions I hadn't written. For example, I asked it to walk me through the route-to-service flow (how `POST /songs/<id>/rate` reaches `notification_service.rate_song`) and to explain what SQLAlchemy's `outerjoin` onto an association table does to the row count. This was genuinely useful for building a mental model of an unfamiliar codebase faster than reading cold.

**Where the AI was wrong, and how I caught it.** For Issue #3 (duplicate songs in search), Claude initially diagnosed a "join fan-out": it claimed the `outerjoin(song_tags)` in `search_songs` would return one row per tag, so a 3-tag song would appear 3 times. I did not take this at face value. I isolated the exact query in `flask shell` and inspected the raw result: for a song with 3 tags (`Crown Heights Anthem`), the query returned `raw rows: 1 | distinct ids: 1`, and a real search returned each song exactly once (`any dup: False`). This falsified the AI's diagnosis — `db.session.query(Song).all()` deduplicates by primary key via SQLAlchemy's identity map, so the `outerjoin` does not surface duplicates through this code path. I then read the full current `search_songs` function myself and confirmed it contains only a single query and a single return, with no "second code path" that the issue hint described. Because I could not reproduce Issue #3 in the code I had, I chose Issue #1 as my third fix instead of documenting a bug I couldn't trigger.

This matched the project brief's warning that asking AI to guess the bug before reading the relevant code "almost always leads you somewhere plausible but wrong." My workflow throughout was: AI helps me understand code I've found → I verify by running it with controlled inputs → I fix only after the reproduction confirms the cause.

---

## Codebase Map

**Architecture.** Mixtape is a JSON API (no HTML frontend — requesting `/` returns 404, which is expected). It follows a strict route → service split: files in `routes/` handle input parsing and response formatting, while all business logic lives in `services/`. Every service function raises `ValueError` when an entity is missing, and every route catches `ValueError` and converts it to a 404 or 400. This is a consistent error contract across the whole app.

**Main files and their roles:**

- `app.py` — Flask application factory (`create_app`). Configures the SQLite database (`mixtape.db`), initializes the `db` object, and registers four blueprints with URL prefixes: `/songs`, `/playlists`, `/users`, `/feed`. Also calls `db.create_all()` inside the app context. The `if __name__ == "__main__"` block calls `create_app()` a second time, which is why running `python app.py` triggers a double-import error and the project requires `FLASK_APP=app:create_app flask run` instead.
- `models.py` — SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables: `friendships` (symmetric user-to-user), `song_tags` (song-to-tag many-to-many), and `playlist_entries` (playlist-to-song, which carries a `position` column so playlist order is explicit, not insertion order). A rating is its own `Rating` model with a unique constraint on `(user_id, song_id)`.
- `routes/songs.py` — song search, detail, rating, and listen endpoints.
- `routes/playlists.py` — playlist creation, detail, song listing, and add-song endpoints.
- `routes/users.py` — user profile, streak, and notification endpoints.
- `routes/feed.py` — "friends listening now" and activity feed endpoints.
- `services/streak_service.py` — listening streak logic (Issue #1 lives here).
- `services/feed_service.py` — friends-listening-now and activity feed logic (Issue #2).
- `services/search_service.py` — song search logic (Issue #3).
- `services/notification_service.py` — notification creation/retrieval; also owns the rate-song and add-to-playlist actions, which fire notifications as a side effect (Issue #4).
- `services/playlist_service.py` — playlist retrieval logic (Issue #5).

**Data flow — rating a song (and its notification side effect):** A user rates a song via `POST /songs/<song_id>/rate`. The route `rate()` in `routes/songs.py` parses `user_id` and `score`, then calls `notification_service.rate_song(user_id, song_id, score)`. Notably, both the rate action and the add-to-playlist action live in `notification_service.py`, not in a song or playlist service — this is because firing a notification is intended to be a side effect of each action. `add_to_playlist()` creates a notification for the song's original sharer; `rate_song()` was supposed to do the same (see Issue #4).

**Pattern noticed:** Routes never contain business logic — they immediately delegate to a service function and only handle parsing and HTTP status codes. The service layer is where every bug in this project lives.

---

## Root Cause Analysis

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:** In `flask shell`, I counted rows directly in the `playlist_entries` join table for the first playlist (7 rows) and compared against what `get_playlist_songs()` returned (6). The endpoint returned exactly one fewer song than the playlist actually contained. Note: I first tried `len(p.songs)` as ground truth but it returned 0 due to how the `lazy="subquery"` relationship loaded, so I switched to counting the join table directly, which gave the honest count of 7.

**How I found the root cause:** I traced from `GET /playlists/<id>/songs` in `routes/playlists.py` to `playlist_service.get_playlist_songs()`. The query itself was correct — it joined `playlist_entries`, filtered by playlist, and ordered by `position` ascending. The problem was the return statement: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice stood out immediately, and the docstring directly above claimed the function "returns all songs in the playlist," which the code contradicted. That contradiction confirmed I'd found the specific cause, not just a suspicious area.

**The root cause:** The return line applied a `[:-1]` slice to the ordered song list. In Python, `[:-1]` returns every element except the last, so the song with the highest `position` (the most recently added) was silently dropped from every playlist. A 7-song playlist returned 6.

**My fix and side-effect check:** I removed the slice — changed `songs[:-1]` to `songs`. After the fix, `get_playlist_songs()` returned all 7 songs. I checked that ordering was preserved by confirming the songs still came back sorted by `position` (first title "Midnight Drive"), since the `.order_by(asc(position))` clause was untouched. I confirmed the returned count now matches the raw join-table count of 7.

*Commit: `fix: return all playlist songs instead of dropping the last one`*

---

### Issue #4 — Notified when a friend adds my song to a playlist, but not when they rate it

**How I reproduced it:** In `flask shell`, I picked a song, identified its sharer, and had a different user rate it via `rate_song()`. I counted the sharer's notifications before and after: both were 1. The rating saved (the function returned a `Rating` object), but no notification was created for the sharer.

**How I found the root cause:** I traced from `POST /songs/<id>/rate` in `routes/songs.py` to `notification_service.rate_song()`. Since the brief said the cause was architectural rather than a typo, I compared `rate_song()` against `add_to_playlist()` in the same file, line by line. `add_to_playlist()` ends with a guarded `create_notification()` call; `rate_song()` validated the score, saved the `Rating`, committed, and returned — with no notification step at all. The entirely missing block, rather than a wrong value, confirmed the root cause.

**The root cause:** `rate_song()` was never wired to notify the song's sharer. Unlike `add_to_playlist()`, it contained no `create_notification()` call, so rating a song silently produced no notification for the person who shared it. This was a missing side effect, not a logic error in existing code.

**My fix and side-effect check:** After the commit in `rate_song()`, I added a notification block mirroring `add_to_playlist()`: a guard (`if song.shared_by != user_id`) so a user isn't notified about rating their own song, followed by a `create_notification()` call with type `song_rated`. After the fix, an outside user's rating raised the sharer's notification count from 1 to 2, with body "darius rated your song 'Midnight Drive' 4 out of 5." Side-effect check: I confirmed the self-rating guard works — a user rating their own song saved the rating but created no notification (count unchanged) — so the fix doesn't notify people about their own ratings and doesn't break the rating path.

*Commit: `fix: notify song sharer when their song is rated`*

---

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:** Rather than waiting for a real Sunday, I isolated `update_listening_streak()` in `flask shell` and fed it controlled inputs. I set a user to a 10-day streak with `last_listened_at` = Saturday June 14, 2025, then called the function with `now` = Sunday June 15 (a genuine consecutive day). The streak reset to 1 instead of incrementing to 11. I confirmed `sunday.weekday()` returned 6.

**How I found the root cause:** I traced from `POST /songs/<id>/listen` → `streak_service.record_listening_event()` → `update_listening_streak()`. Reading the increment branch, the condition was `elif days_since_last == 1 and today.weekday() != 6:`. The `today.weekday() != 6` clause was the tell — it had nothing to do with the documented streak rules, which say only that consecutive days increment and skipped days reset. Nothing in the docstring mentions weekdays, so this extra condition had no legitimate reason to exist.

**The root cause:** Python's `datetime.weekday()` returns 6 for Sunday. The increment branch required `days_since_last == 1 and today.weekday() != 6`, so on Sundays the branch failed even for a valid consecutive-day listen, and execution fell through to the `else` clause, which resets the streak to 1. Any streak that continued into a Sunday was wiped regardless of actual listening behavior.

**My fix and side-effect check:** I removed the ` and today.weekday() != 6` clause so the branch is simply `elif days_since_last == 1:`. After the fix, a Saturday→Sunday listen incremented the streak from 10 to 11. Side-effect check on both sides of the boundary: (1) a non-Sunday consecutive day (Monday→Tuesday) still increments to 11, and (2) a genuine 5-day gap still resets to 1. So the fix restores Sunday increments without disabling the legitimate reset-on-skip behavior.

*Commit: `fix: remove erroneous Sunday condition from streak increment`*

---

### Regression Test

The test suite includes test_streak_increments_on_sunday in tests/test_streaks.py, which sets up a Saturday listen followed by a Sunday listen and asserts the streak increments to 2. Under the original bug (the and today.weekday() != 6 condition in update_listening_streak), this test fails because the Sunday listen resets the streak to 1 instead of incrementing it. After my fix, it passes.

I verified this by running pytest tests/ — all 13 tests pass, confirming the fix resolves Issue #1 without breaking the other streak behaviors covered by the suite (new-user start at 1, same-day no-double-count, consecutive-day increment, and reset-after-skipped-day). This test would have caught Issue #1 before it was introduced, since any reintroduction of the Sunday condition causes it to fail.

*Note:* this test was present in the starter repo. I did not author it, but I verified that it functions as a regression test for the bug I fixed by confirming it fails under the original condition and passes after the fix.

---

## Git Log

<img width="935" height="105" alt="Screenshot 2026-07-04 at 11 42 33 PM" src="https://github.com/user-attachments/assets/9bdb221b-28e0-4270-a37e-a1e540035810" />

