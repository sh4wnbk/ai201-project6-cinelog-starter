# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the `verb_to_noun` convention the reviewer pointed to (`add_to_collection()`, `remove_from_collection()`, `get_collection()` in `services/collection_service.py`).

**How I verified:** Ran `grep -rn "save_to_watchlist" --include="*.py" .` across the whole repo before renaming to find every call site — it turned up exactly two hits: the definition in `services/watchlist_service.py:12` and one call site in `routes/watchlist/watchlist.py` (both the import and the call inside `add_film()`). Updated both. Re-ran the same grep after the change and it returned nothing, confirming no stale references. Also ran `pytest tests/ -v` (all 4 existing collection tests still pass, since this rename doesn't touch that module) and did a quick `create_app()` import smoke test to confirm the route module still imports cleanly with the new name.

## Comment 2 — Deduplication
**What I did:** Looked at `add_to_collection()` in `services/collection_service.py` first — it queries `CollectionEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` before inserting and raises a dedicated `AlreadyInCollectionError` if a row already exists, rather than silently ignoring the duplicate or letting a DB integrity error bubble up. I followed the exact same shape for the watchlist: added an `AlreadyInWatchlistError` exception class and, inside `add_to_watchlist()`, a `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` check that raises before any insert happens. Also updated `routes/watchlist/watchlist.py` to catch this new error and return `409`, matching how `routes/collection.py` handles `AlreadyInCollectionError` (and added a `404` handler for `FilmNotFoundError` in the same route, which the route was missing before).

**How I verified:** Wrote a quick manual script (`add_to_watchlist` twice with the same user/film) confirming the second call raises `AlreadyInWatchlistError` instead of creating a second row, and that a call with a made-up `film_id` still raises `FilmNotFoundError`. Also re-ran `pytest tests/ -v` to confirm the existing collection tests are unaffected. Formal `test_watchlist.py` coverage for this comes in Comment 3.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, using the exact same fixture structure as `tests/test_collection.py` (`app`, `sample_user`, `sample_film` — same in-memory SQLite setup and teardown). Modeled `test_add_to_watchlist_nonexistent_film_raises` directly on `test_add_to_collection_nonexistent_film_raises`: it uses the same fake-UUID literal (`"00000000-0000-0000-0000-000000000000"`) and asserts `FilmNotFoundError` is raised. Per `CONTRIBUTING.md`'s rule that any new service function needs a happy-path test, a duplicate/conflict test, and a nonexistent-ID test, I also added `test_add_to_watchlist_creates_entry` (happy path) and `test_add_to_watchlist_duplicate_raises` (verifies the Comment 2 dedup fix) in the same file, mirroring `test_collection.py`'s equivalent three tests.

**How I verified:** Ran `pytest tests/test_watchlist.py -v` — all 3 tests pass — then `pytest tests/ -v` for the full 7-test suite (existing collection tests + new watchlist tests), all green.

## Stretch — remove_from_watchlist()
**What I did:**
**How I verified:**

## Stretch — Second test
**What I did:**
**Why this edge case:**

## Stretch — Visibility toggle endpoint
**What I did:**

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## Commit History

<!-- git log --oneline screenshot goes here -->

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
