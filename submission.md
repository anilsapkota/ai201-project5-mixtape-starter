### Issue #1: My listening streak keeps resetting

**Issue number and title:** Issue #1 — My listening streak keeps resetting

**How you reproduced it:**
I traced the entry point first: `POST /songs/<song_id>/listen` in `routes/songs.py`
calls `record_listening_event()` in `streak_service.py`, which is the function that
actually updates the streak. However, `record_listening_event()` hardcodes
`now = datetime.now(timezone.utc)` internally, so hitting the HTTP endpoint gives
no control over which calendar day is being tested. Instead, I called the lower-level
function it delegates to, `update_listening_streak(user, now)`, directly in
`flask shell`, since it accepts an explicit `datetime` as an argument. Using the
seeded user `nova` (streak of 7, `last_listened_at` earlier the same day), I called
`update_listening_streak(user, saturday)` with `saturday = datetime(2026, 7, 4, 12, 0, 0, tzinfo=timezone.utc)`,
then `update_listening_streak(user, sunday)` with `sunday = datetime(2026, 7, 5, 12, 0, 0, tzinfo=timezone.utc)`
— a real, consecutive Saturday/Sunday pair. The streak dropped to 1 after the Sunday
call instead of incrementing to 8, confirming the bug.

**How you found the root cause:**
I read `update_listening_streak()` in `streak_service.py` branch by branch against
its own docstring, which lists four rules and never mentions weekdays at all. The
`elif` branch responsible for incrementing on a one-day gap reads
`days_since_last == 1 and today.weekday() != 6`. I confirmed by running
`datetime(2026, 7, 5).weekday()` directly in `flask shell` that it returns `6` —
Python's `.weekday()` convention numbers Monday as `0` through Sunday as `6`. Plugging
that into the condition by hand (`True and (6 != 6)` → `True and False` → `False`)
showed the branch fails to fire on a Sunday, falling through to the `else` reset
branch instead. That was the moment of confidence: it wasn't a vague "something's
off with dates," it was a specific, provable value (`6`) causing a specific,
traceable branch to be skipped.

**The root cause:**
The `elif` branch that's supposed to increment the streak on a one-day gap also
required `today.weekday() != 6`, which excludes Sunday. Since Sunday maps to
weekday `6` under Python's `.weekday()` convention, any streak update landing on
a Sunday gets routed to the `else` branch instead, resetting the streak to 1
instead of incrementing it — even though exactly one day had passed, same as any
other day of the week. Nothing in the function's own documented rules calls for
this weekday exception.

**Your fix and side-effect check:**
Removed the `and today.weekday() != 6` clause, leaving
`elif days_since_last == 1: user.listening_streak += 1`, which matches all four
documented rules with no weekday exception. Verified with a fresh Saturday/Sunday
pair (July 11–12, 2026, after resetting past nova's earlier test state): the
Saturday call correctly reset the streak to 1 (6-day gap), and the Sunday call
correctly incremented it to 2 — confirming the fix works through the same code
path used to reproduce the bug, not just in isolation.


### Issue #5: The last song in a playlist never shows up

**Issue number and title:** Issue #5 — The last song in a playlist never shows up

**How you reproduced it:**
I traced the entry point first: `GET /playlists/<playlist_id>/songs` in
`routes/playlists.py` calls `get_playlist_songs(playlist_id)` in
`playlist_service.py`. I found a real playlist ID in `flask shell` via
`Playlist.query.first().id`, then ran `get_playlist_songs()` on it directly and
got `len(songs) == 6`. To confirm that was actually wrong (not just assume it),
I independently counted how many songs were really linked to that playlist by
querying the `playlist_entries` association table directly, bypassing the
function under test entirely: `db.session.query(playlist_entries).filter(...).count()`
returned `7`. The 6-vs-7 mismatch confirmed the bug — exactly one song was
missing from every playlist's result.

**How you found the root cause:**
I read `get_playlist_songs()` in full, including its docstring, before forming
any theory. The docstring's `Note:` block explicitly states "This function
returns all songs in the playlist," directly above a `return` statement that
does `[song.to_dict() for song in songs[:-1]]`. The query building `songs` above
that line is correct — it joins `playlist_entries`, filters by playlist, and
orders by `position` ascending, so it already fetches every song in the right
order. The contradiction between the docstring's explicit claim and the slice
on the very next meaningful line was the moment of confidence: this wasn't a
query bug, it was a slicing bug applied after the correct data was already fetched.

**The root cause:**
The final line of `get_playlist_songs()` returns `songs[:-1]` instead of `songs`.
In Python, `list[:-1]` returns every element except the last one. Since `songs`
already holds the complete, correctly-ordered result of the query, this slice
unconditionally drops the last song (by position) from every playlist's result,
regardless of how many songs the playlist actually has — a stray slice applied
after correct data retrieval, not a flaw in the query itself.

**Your fix and side-effect check:**
Changed `songs[:-1]` to `songs`. Verified by re-running the same two checks from
reproduction after restarting `flask shell` (to avoid testing stale, already-imported
code): `get_playlist_songs()` now returns `len(songs) == 7`, matching the independent
`playlist_entries` count of `7`. I also reasoned through the empty-playlist edge case
by hand: `[][:-1]` and `[]` both evaluate to `[]`, so a playlist with zero songs was
never affected by this bug either way — no regression risk there.

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

**Issue number and title:** Issue #4 — Missing notification when a friend rates your song

**How you reproduced it:**
I traced two entry points since the bug report describes two different actions:
`POST /songs/<song_id>/rate` (`routes/songs.py`) → `notification_service.rate_song()`,
and `POST /playlists/<playlist_id>/songs` (`routes/playlists.py`) →
`notification_service.add_to_playlist()`. Using `flask shell`, I found a real song
shared by `nova` and a different user, `darius`, then recorded a baseline
notification count for `nova` (`Notification.query.filter_by(user_id=nova.id).count()`
→ `1`). I called `rate_song(user_id=darius.id, song_id=<nova's song>, score=5)`
directly and re-checked the count — it stayed at `1`, unchanged, confirming no
notification was created even though a different user rated the song.

**How you found the root cause:**
I read `add_to_playlist()` and `rate_song()` in `notification_service.py` side by
side, since the bug report implies one of the two actions works correctly
(playlist-add) and one doesn't (rating). `add_to_playlist()` ends with a clear
pattern: check `if song.shared_by != added_by_user_id`, then call
`create_notification(...)`. `rate_song()` has no equivalent block anywhere —
it upserts the `Rating`, commits, and returns. I also checked
`create_notification()`'s own docstring, which explicitly lists `'song_rated'`
as an example valid `notification_type` — confirming this notification type was
intended to exist from the start, it just was never wired up anywhere in the
codebase. That combination (a working reference pattern + a completely absent
equivalent + a docstring naming the missing type) was the confirming evidence,
not just a guess.

**The root cause:**
`rate_song()` never calls `create_notification()` at all. This is a missing
step, not an incorrect condition — the notification pattern used correctly by
`add_to_playlist()` (check ownership, then notify the sharer) was simply never
implemented for the ratings flow when it was built.

**Your fix and side-effect check:**
Added the missing block to `rate_song()`, mirroring `add_to_playlist()`'s pattern
exactly: after `db.session.commit()`, if `song.shared_by != user_id`, call
`create_notification(user_id=song.shared_by, notification_type="song_rated", body=f"{rater.username} rated your song '{song.title}' {score}/5.")`.
Verified via `flask shell` after restarting the session (to avoid running stale,
already-imported code): calling `rate_song()` with a rater different from the
song's

## AI Usage

I used Claude as a guided pairing partner rather than to generate answers directly —
for each bug, Claude asked me to find the entry point, form a hypothesis, and verify
it myself before confirming anything, only stepping in when I was genuinely stuck.

**What I asked it to explain/trace:** For each bug, I had it help me confirm which
route file and endpoint actually called into the relevant service function, since
the assignment discourages skipping straight to the service file. For Issue #4, I
specifically asked it to compare `add_to_playlist()` and `rate_song()` side by side,
since the bug wasn't a wrong condition but a missing block, and reading the broken
function alone never would have surfaced that on its own.

**What it helped me understand:** Python's `.weekday()` convention (Monday=0 ...
Sunday=6, versus the separate `isoweekday()` method which numbers differently) —
I initially assumed the fix for Issue #1 was changing `!= 6` to `!= 7`, which
Claude had me check against the actual `.weekday()` range before I made that
mistake in the code. It also walked me through what Python list slicing
(`songs[:-1]`) actually does, which was the entire root cause of Issue #5.

**Where I had to verify or push back myself, or got tripped up:**
- I hit a stale-code trap twice — after editing `streak_service.py` and later
  `playlist_service.py`, I re-ran tests in the same `flask shell` session and got
  the *old* buggy behavior back, because `flask shell` only imports code once at
  startup. I had to learn to `exit()` and restart the shell after every code change
  before re-testing.
- For Issue #1, I initially tried reusing a Saturday/Sunday date pair from an
  earlier test without checking that it was still *after* the user's current
  `last_listened_at` in the database — Claude had me check the actual current
  state first rather than assume it, which caught the mistake before I ran it.
- I ran into PowerShell vs. Git Bash syntax differences trying to set
  `FLASK_APP` inline, and got a red herring 404 on the bare `/` root URL that
  turned out to just be an unregistered route, not an actual problem.
- For Issue #5, I verified the fix by independently counting rows in the
  `playlist_entries` table directly, rather than trusting the same function I
  was testing — that's what gave me real proof (6 vs. 7) instead of just an
  assumption that my read of the code was correct.
- I did not end up verifying every edge case for every bug (e.g. Issue #4's
  self-rating guard and repeated-rating behavior weren't separately tested in
  this session) — flagging that honestly rather than claiming full coverage.

  ## Codebase Map

**Main files and their roles:**
- `app.py` — Flask application factory. Registers four blueprints (`songs`,
  `playlists`, `users`, `feed`) and initializes the SQLAlchemy `db` object.
- `models.py` — All SQLAlchemy models: `User`, `Song`, `Tag`, `ListeningEvent`,
  `Rating`, `Playlist`, `Notification`, plus association tables for friendships,
  song tags, and playlist entries (the last one has an explicit `position` column,
  so playlist order is stored data, not just insertion order).
- `routes/` — Thin controllers. Every route I looked at (`songs.py`, `playlists.py`,
  `users.py`) parses the request and immediately calls exactly one function in
  `services/`. No business logic lives in the routes themselves.
- `services/streak_service.py` — `record_listening_event()` is the entry point
  called from `POST /songs/<id>/listen`; it hardcodes the current time internally
  and delegates the actual streak math to `update_listening_streak(user, now)`,
  which is a pure function I could call directly with a controlled datetime to
  test specific day transitions without waiting for the calendar.
- `services/playlist_service.py` — `get_playlist_songs()` (called from
  `GET /playlists/<id>/songs`) correctly queries and orders songs by `position`,
  but had a stray slice on the final return line.
- `services/notification_service.py` — `create_notification()` is the generic
  helper. `add_to_playlist()` (called from `POST /playlists/<id>/songs`) follows
  the correct pattern: perform the action, then notify the original sharer if
  they weren't the one who acted. `rate_song()` (called from
  `POST /songs/<id>/rate`) performed the action but never called the notify step.

**Data flow — rating a song vs. adding to a playlist:**
Both actions are meant to notify the song's original sharer, but only route
through similarly-shaped functions in the same file. Comparing them directly
(rather than reading either one in isolation) is what exposed that one had a
step the other was missing entirely.

**Pattern noticed:** Every bug I found was located either by (1) a contradiction
between a function's own docstring and its actual code, or (2) comparing a
working code path against a structurally similar broken one. Neither required
guessing — both just required reading carefully and having a reference point.