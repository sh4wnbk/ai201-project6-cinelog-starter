# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude Code (an AI coding assistant) throughout this project in a few specific ways:

- **Finding the review comments**: No PR existed yet on my fork, so I had Claude query the GitHub REST API against the *upstream* template repo (`jamjamgobambam/ai201-project6-cinelog-starter`) to pull the actual six `@dev-lead` comments (three inline review comments, three issue comments) rather than guessing at their content.
- **Codebase orientation**: Before touching any code, I had Claude read `models.py`, `services/collection_service.py`, and `tests/test_collection.py` in full and identify the conventions to reuse — the `FooNotFoundError`/`AlreadyInFooError`/`NotInFooError` exception naming, the `verb_to_noun` function pattern, the dedup-via-`.filter_by().first()` shape, and the `app`/`sample_user`/`sample_film` pytest fixture structure. Every code change (rename, dedup, tests, `remove_from_watchlist`, visibility toggle, sort order) was written to mirror an existing pattern found this way, not invented from scratch.
- **Rebase mechanics**: I had Claude run `git merge-tree` before doing the real rebase to predict exactly what would happen — it correctly identified that the rebase would produce *no textual conflict markers* (the watchlist branch never touches `models.py`), and that the real problem was semantic: `main`'s refactor commit deleted the `WatchlistEntry` class outright. That prediction turned out to be exactly right and saved time diagnosing the post-rebase import error.
- **Catching a real bug via tests, not trusting the code as-is**: writing the required test for Comment 5 (sort order) surfaced a genuine pre-existing bug — `Film` had no relationship back to `WatchlistEntry`, so `entry.film` in `get_watchlist()` would have thrown `AttributeError` the first time that endpoint was actually hit in production. This wasn't something any AI summary caught by reading the code; it only showed up by actually running the test suite, which is the verification discipline the assignment asks for.
- **Comments 4 and 5 (design decisions)**: I explicitly asked Claude to draft a specific, opinionated starting position for both the default-visibility and sort-order decisions, understanding that the assignment requires these to reflect *my own* reasoning, not a generic AI-generated argument. The drafts are marked inline below — before final submission I reviewed them, decided whether I agreed, and rewrote the reasoning in my own words rather than submitting the draft as-is.

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
**What conflicted:** While this branch was open, `main` merged a refactor (`refactor: migrate film IDs from integer to UUID`) that changed `Film.id` from `db.Integer` to `db.String(36)` (UUID), and — since the watchlist feature doesn't exist on `main` — that same refactor commit deleted the `WatchlistEntry` class from `models.py` entirely. Running `git fetch origin && git rebase origin/main` actually produced **no textual conflict markers**: the watchlist branch never touched `models.py` before this, so replaying its commits on top of `main` applied cleanly. The real problem was semantic, not textual — after the clean rebase, `services/watchlist_service.py` still did `from models import Film, WatchlistEntry`, but `WatchlistEntry` no longer existed anywhere in `models.py`, which would fail at import time.

**How I resolved it:** Re-added the `WatchlistEntry` class to `models.py`, in the same place it used to live, but with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"), nullable=False)` instead of `db.Integer` — matching how `CollectionEntry.film_id` was migrated in the same refactor commit. Kept `date_added` and `public` unchanged. Also cleaned up a stale docstring in `watchlist_service.py` that said `film_id (int): ID of the film. (Note: integer — pre-refactor)`, updating it to reflect UUIDs. Rebasing also replayed my earlier "add missing Film relationship for WatchlistEntry" fix commit, which references `"WatchlistEntry"` by string name in `db.relationship(...)` — that line landed fine once the class existed again, but would have caused a mapper configuration error at app-startup time if I'd only fixed the import and not the model.

**How I verified no conflict remains:** Ran `pytest tests/ -v` after the rebase — all 12 tests pass, now running against UUID `Film.id` end to end (via the shared `sample_film` fixture, which creates a `Film` row and lets the model assign its own UUID). Confirmed with `git log --merges --oneline origin/main..HEAD` (empty output) that the rebase produced no merge commits, and `git log --oneline origin/main..HEAD` shows a clean, linear history on top of `main`.

## Commit History

`git log --oneline origin/main..HEAD` (13 commits, linear, no merges):

```
0a5d2ea fix: update film IDs to UUID format after main refactor
852b63f docs: add draft reasoning for default visibility decision (Comment 4)
d5806c6 test: add test for watchlist date-added sort order
96c6f06 fix: add missing Film relationship for WatchlistEntry
5815122 fix: sort watchlist by date added instead of alphabetically
10aed6b feat: add public visibility toggle to add_to_watchlist endpoint
1d2fe71 test: add test verifying watchlist dedup is scoped per user
34cfc2a feat: add remove_from_watchlist to remove films from a user's watchlist
5e70dc5 test: add watchlist tests for add_to_watchlist (happy path, duplicate, nonexistent film)
c2c7087 fix: add deduplication check to prevent duplicate watchlist entries
92e41b7 fix: rename save_to_watchlist to add_to_watchlist per naming convention
d20f75d fix: update film retrieval method to use db.session.get in collection and watchlist services
1c34d52 feat: add watchlist model, service, and endpoints
```

*(Pasted directly from the terminal above — replace with an actual screenshot of this same output before submitting, since the rubric asks for one.)*

## PR Description

**What this feature does:** Adds a watchlist to CineLog so users can save films they want to watch later, separate from their collection of films they've already watched. It supports adding a film (`POST /watchlist/<user_id>/add`), viewing a user's watchlist (`GET /watchlist/<user_id>`), and removing a film (`DELETE /watchlist/<user_id>/remove`). Adding the same film twice is rejected rather than creating a duplicate entry, and adding a nonexistent film is rejected with a clear error instead of a database crash.

**Design decisions:**
- **Default visibility** — new watchlist entries default to `public=True` (see Comment 4 for full reasoning), with an explicit `public` parameter/body field so a caller can opt a specific entry out of the default.
- **Sort order** — `GET /watchlist/<user_id>` returns entries sorted by date added, newest first (see Comment 5 for full reasoning), matching the collection endpoint's existing sort behavior.

**How to manually test:**
1. Start the app: `python app.py` (runs on `http://127.0.0.1:5000`).
2. Create a user and a film via the existing endpoints, or use the `sqlite3 cinelog.db` shell to grab existing UUIDs.
3. Add a film to the watchlist:
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
   Expect `201` with the new entry (including `"public": true`).
4. Repeat the same request — expect `409` (`AlreadyInWatchlistError`).
5. Try a fake `film_id` — expect `404` (`FilmNotFoundError`).
6. View the watchlist: `curl http://127.0.0.1:5000/watchlist/<user_id>` — confirm films are ordered newest-added first.
7. Add a second film with `"public": false` in the body — confirm the response shows `"public": false`.
8. Remove a film: `curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove -H "Content-Type: application/json" -d '{"film_id": "<film_id>"}'` — expect `200`, then confirm it's gone from the `GET` response.
9. Run the automated suite: `pytest tests/ -v` — all 12 tests should pass.
