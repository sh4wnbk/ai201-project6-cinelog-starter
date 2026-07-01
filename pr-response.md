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
**What I did:** Added `remove_from_watchlist(user_id, film_id)` to `services/watchlist_service.py`, following `remove_from_collection()`'s exact pattern in `services/collection_service.py`: look up the entry by `(user_id, film_id)`, raise a dedicated error (`NotInWatchlistError`, new here, mirroring `NotInCollectionError`) if it doesn't exist, otherwise delete and commit and return `True`. Added a matching `DELETE /watchlist/<user_id>/remove` route in `routes/watchlist/watchlist.py`, mirroring `routes/collection.py`'s remove endpoint (same body shape, same 404-on-missing behavior).

**How I verified:** Added `test_remove_from_watchlist_removes_entry` (confirms the row is actually gone from the DB after removal, not just that no exception was raised) and `test_remove_from_watchlist_not_in_watchlist_raises` (confirms removing a film that was never added raises `NotInWatchlistError` rather than silently no-op'ing). Both pass as part of the full `pytest tests/ -v` run (9/9 passing).

## Stretch — Second test
**What I did:** Added `test_add_to_watchlist_same_film_different_users_both_succeed`, which has two different users add the same film to their (separate) watchlists and asserts both succeed and both rows exist.

**Why this edge case:** Comment 2 asked for a dedup check, and the tests in Comment 3 confirm dedup rejects a *repeat* add by the *same* user. But nothing in the existing tests pins down what the dedup key actually is — it would be easy to accidentally implement this as "a film can only be on one watchlist, period" (e.g. a unique constraint on `film_id` alone) instead of "a user can't add the same film twice" (unique on `(user_id, film_id)`). Since watchlists are personal by design, silently blocking a second user from watchlisting a popular film would be a real, easy-to-miss bug that the required tests wouldn't catch. This test locks in the correct scope of the dedup logic added in Comment 2.

## Stretch — Visibility toggle endpoint
**What I did:** Added a `public` keyword parameter to `add_to_watchlist(user_id, film_id, public=True)`, passed straight through to the `WatchlistEntry` constructor. `POST /watchlist/<user_id>/add` now reads `public` from the JSON body via `data.get("public", True)` — if the caller omits it, behavior is unchanged (defaults to `True`, same as the model's own column default); if they pass `false`, that value is respected on the created entry. This lets a caller set visibility explicitly at creation time instead of always inheriting the model default, which is what Comment 4 asked for. Added `test_add_to_watchlist_public_false_is_respected` to confirm the override actually persists.

## Comment 4 — Default visibility
**My position:** [DRAFT — rewrite in your own words before submitting] Keep `public=True` as the default on `WatchlistEntry`.

**Reasoning:** [DRAFT] CineLog's collection feature (already-watched films, with ratings) is the app's existing shareable surface — `CollectionEntry` has no visibility flag at all, which implicitly means "always visible." A watchlist is the natural extension of that same social/discovery loop: it's the mechanism by which a user's friends find out "they want to watch Dune 2, I should recommend the extended cut" or notice a shared interest before either has actually watched anything. If watchlists defaulted to private, that discovery loop mostly wouldn't fire, because defaults are sticky — the research on opt-in vs. opt-out settings is consistent that most users never touch a settings toggle regardless of how they feel about the underlying tradeoff, so whatever CineLog picks as the default *is*, in practice, the behavior for the overwhelming majority of watchlist entries. Given that the collection feature already treats "logged film activity" as public-by-default, keeping the watchlist consistent avoids a confusing app where one type of film activity is shared and a closely related one silently isn't.

**Tradeoff acknowledged:** [DRAFT] The real cost of `public=True` is that a watchlist reveals *intent*, not history — it's more exposing than a collection entry, because it's a list of things a user hasn't actually watched yet and may feel differently about once they see the review comments' judgmental cousin: "why do you want to watch that." A private-by-default watchlist would better protect users who are self-conscious about genre choices or who use the watchlist as a personal scratchpad rather than a public statement. I think that's a legitimate concern, which is why I implemented the `public` parameter on `add_to_watchlist()` (stretch feature) so a user — or a future settings UI — can opt a specific entry out of the default without us having to sacrifice the discovery behavior for everyone else.

**(Note to self before submitting):** Same as Comment 5 — this is a starting draft, not your final answer. Decide if `public=True` is actually the position you'd defend, and rewrite the reasoning to reflect your own read of CineLog's users before submitting.

## Comment 5 — Sort order
**My position:** [DRAFT — rewrite in your own words before submitting] I agree with the maintainer: `get_watchlist()` should sort by `date_added` descending (newest first), not alphabetically by title. Changed `order_by(Film.title.asc())` to `order_by(WatchlistEntry.date_added.desc())`.

**Reasoning:** [DRAFT] A watchlist is a queue of intent, not a reference catalog. `get_collection()` already sorts by `date_added.desc()` for exactly this reason — collection entries represent things you've *done*, and users want to see what they logged most recently at the top. The watchlist is the same shape of data: an ordered log of "I want to watch this," not an alphabetized index a user is trying to look something up in. Alphabetical order actively works against the primary use case (a user opening their watchlist to decide what to watch next probably cares about recency/intent, not the letter a title starts with), and it's inconsistent with the rest of the app's sort conventions for no clear reason.

**Engagement with reviewer's point:** [DRAFT] The maintainer's reasoning — "most users want to see what they added recently" — is the same argument that already justifies `get_collection()`'s sort order, so I don't think there's a case for the watchlist to be the odd one out here. The one place I'd push back if given the choice: if CineLog ever adds a "long watchlist" power-user flow (someone with 200+ saved films trying to find one specific title), alphabetical sort becomes valuable again as a lookup mechanism — at that point I'd advocate for a client-side or query-param sort toggle rather than reversing the default, since the default should optimize for the common case, not the power-user edge case.

**(Note to self before submitting):** This whole section is my draft to get you started — the assignment explicitly wants *your* reasoning here, not AI-generated argument. Read it, decide if you actually agree, and rewrite it in your own voice/logic before this goes in the real submission.

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## Commit History

<!-- git log --oneline screenshot goes here -->

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
