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


