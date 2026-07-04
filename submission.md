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