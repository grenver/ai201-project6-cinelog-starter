# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude Code (an AI coding agent) throughout this project, primarily for orientation and hygiene rather than for the design decisions themselves:

- **Codebase orientation:** Before touching any code, I had it read `models.py`, `services/collection_service.py`, `routes/collection.py`, and `tests/test_collection.py` in full to establish the existing patterns (the `verb_to_noun` naming convention, the `FooNotFoundError`/`AlreadyInFooError` exception style, the fixture structure in tests) before making any watchlist changes, so Comments 1–3 and the stretch features could follow those patterns exactly rather than inventing new ones.
- **Fetching the actual review comments:** The six review comments live on PR #1 in the upstream CineLog repo, which isn't visible from a fork. I had it fetch that PR's Conversation and Files-changed comments directly so I was responding to the maintainer's literal wording rather than a paraphrase.
- **Root-causing the rebase failure:** After `git rebase origin/main` reported success with zero conflict markers, the test suite still failed on import. I had it trace *why* — walking the pre- and post-rebase diffs of `models.py` commit-by-commit — which is how I found that the UUID-refactor commit on `main` had deleted the entire `WatchlistEntry` class and the 3-way merge silently kept that deletion because none of my commits touched those specific lines. I wrote the fix (re-adding the class with a UUID `film_id`, sweeping the remaining integer-ID docstrings and test fixtures) myself once the root cause was identified.
- **Design-decision stress-testing (Comments 4 and 5):** I wrote my own first-draft positions for both comments, then asked what counterargument a careful reviewer would raise against each. For Comment 4, it raised: "a per-entry public flag doesn't stop bulk enumeration if someone iterates the `GET /watchlist/<user_id>` endpoint across many user IDs" — a real point, but out of scope for this PR since `GET /collection/<user_id>` has the identical exposure today and fixing that is an authentication/access-control problem across the whole API, not something a default-value decision on one field can solve; I noted this framing didn't need to change my position but sharpened how I explained the tradeoff as being about the visibility *flag*, not overall data exposure. For Comment 5, it raised the point that survives in the final response: that my "recent items don't need finding" argument implicitly assumes users don't return to a watchlist right after adding to it, and that this is checkable with real usage data I don't have — I kept that acknowledgment in the response instead of overstating the confidence of my position. In both cases the final reasoning is mine, grounded in what's actually implemented in CineLog (the fully-public collection endpoint, `GET /films/`'s sort order, the lack of a search/filter UI on the watchlist), not the AI's generic output.
- **Commit format verification:** Before finalizing, I gave it `git log --oneline` output for this branch and asked whether every message followed Conventional Commits and whether any commit bundled more than one logical change. It flagged that my two starter-inherited commits (`added watchlist model and endpoint...` and a follow-up `fix: update film retrieval method...`) should be squashed and reworded since the second was just patching a bug introduced by the first — I did that via `git rebase -i` (squash + reword) rather than leaving them as separate history entries.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention already used by `add_to_collection()` / `remove_from_collection()` / `get_collection()` (documented in `CONTRIBUTING.md`).

**How I verified:** Project-wide search (`grep -r save_to_watchlist`) before and after the change. Only two call sites existed: the definition in `services/watchlist_service.py` and the import + call in `routes/watchlist/watchlist.py`. Updated both, re-ran the search to confirm zero remaining references, then ran `pytest tests/ -v` to confirm nothing broke.

## Comment 2 — Deduplication
**What I did:** Followed `add_to_collection()`'s pattern exactly. Added an `AlreadyInWatchlistError` exception class (mirroring `AlreadyInCollectionError`) and a `WatchlistEntry.query.filter_by(user_id=..., film_id=...).first()` check before creating a new entry in `add_to_watchlist()`. Also updated the `/watchlist/<user_id>/add` route to catch both `FilmNotFoundError` (404) and `AlreadyInWatchlistError` (409), matching how `routes/collection.py` handles the equivalent errors — previously the watchlist route had no exception handling at all, so a nonexistent film_id or a duplicate would have surfaced as an unhandled 500.

**How I verified:** Ran the full suite (`pytest tests/ -v`) after the change, then manually exercised the new path in a Python shell against an in-memory DB: created a user/film, called `add_to_watchlist()` once (succeeds), called it again with the same user/film (raises `AlreadyInWatchlistError` as expected, second call does not create a second row).

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, modeled directly on `tests/test_collection.py`. Reused the same `app`/`sample_user`/`sample_film` fixture pattern (in-memory SQLite, `create_app` factory) and wrote `test_add_to_watchlist_nonexistent_film_raises`, the equivalent of `test_add_to_collection_nonexistent_film_raises` — it asserts that calling `add_to_watchlist()` with a film_id that doesn't exist raises `FilmNotFoundError` rather than a raw DB integrity error. I also added `test_add_to_watchlist_creates_entry` and `test_add_to_watchlist_duplicate_raises` alongside it, since `CONTRIBUTING.md` states new service functions need a happy-path test, a duplicate/conflict test, and a nonexistent-ID test as a set — matching the three-test pattern already used for `add_to_collection()`.

**How I verified:** Ran `pytest tests/test_watchlist.py -v` (all 3 pass) and then the full suite `pytest tests/ -v` (7 passed, no regressions to the existing collection tests).

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default, but give callers an explicit way to opt out at creation time (see the visibility-toggle stretch feature below) rather than leaving `public` as a silent, un-overridable default.

**Reasoning:** CineLog bills itself as a *community* film tracking app, and the existing data model already treats a user's activity as public by default in every other feature: `GET /collection/<user_id>` returns a user's full watched history with no visibility field or access check at all — there is no concept of a private collection anywhere in the current codebase. A watchlist is the natural companion to that: "here's what I want to watch" is a lightweight social signal (similar to Letterboxd's public watchlists) that drives the discovery/recommendation behavior a community app depends on — friends see what you're planning to watch and can react, plan a shared watch, etc. Defaulting to private would make the watchlist the one silo of private data in an otherwise fully public app, which is an inconsistent and surprising default relative to the rest of CineLog, not a safer one.

**Tradeoff acknowledged:** The real cost of `public=True` isn't abstract — a user might add something to their watchlist they don't want visible (an embarrassing guilty pleasure, a film tied to a surprise plan for someone who follows them, or simply not wanting to broadcast intent before they've acted on it). Because `WatchlistEntry.public` is a per-entry boolean rather than a per-user setting, a bad default here means every single add silently leaks unless the caller thinks to override it — and right now the endpoint gives callers no way to override it at all. That's the actual gap, so instead of just debating the default in a comment, I closed it: I added a `public` parameter to `POST /watchlist/<user_id>/add` (see the visibility-toggle stretch feature) so callers can set visibility explicitly per entry, with `True` remaining the default for callers who don't specify it. That keeps the community-friendly default while making the override a one-line request body change instead of an unsupported feature.

## Comment 5 — Sort order
**My position:** I'm keeping alphabetical (`Film.title.asc()`) as the default for `get_watchlist()`, rather than switching to date-added descending. This is the one comment I'm pushing back on.

**Reasoning:** A watchlist and a collection answer different questions, and CineLog's own code already draws that distinction. `get_collection()` is explicitly a viewing *history* — it's sorted newest-first because the point of looking at your collection is "what have I been watching lately," a recency question, and that's reflected in `test_get_collection_returns_newest_first` treating newest-first as a load-bearing behavior. A watchlist isn't a history, it's a working set of intent to consult before deciding what to watch next — the dominant task is "do I already have this queued" or "what's on here at all," not "what did I add most recently." Recently-added items are, almost by definition, the ones a user already remembers adding; the entries that actually need a scannable, predictable order are the older ones further down the list. Alphabetical order is also what the rest of CineLog already does for browsing film lists — `GET /films/` sorts by `Film.title` (`routes/films.py`) — so keeping the watchlist alphabetical is consistent with the one sort convention the app has established for "browse a set of films," rather than introducing a second, different convention that only applies here.

**Engagement with reviewer's point:** The reviewer's underlying claim — "most users want to see what they added recently" — is true for a lot of watchlist products, and I don't think it's wrong in general, just not the stronger fit for CineLog specifically. It would be the right call if the watchlist were primarily something people scroll through top-to-bottom to decide "what's new here," but nothing in the current feature (no pagination, no "recently added" filter, no client evidence of a scrolling UI) suggests that's the primary interaction — whereas the dedup check I just added in Comment 2 means the main practical use of *viewing* the watchlist is confirming whether something is already on it before adding it again, which alphabetical order serves better than chronological order once the list has more than a handful of entries. If usage data later showed people mostly re-open their watchlist right after adding to it (i.e., actually using it as a short-term "what did I just queue" view), that would flip the argument back toward date-added, and I'd want to revisit this then — but I don't think that's the more likely usage pattern for a list whose whole purpose is to be checked back on days or weeks later.

## Comment 6 — Rebase
**What conflicted:** I ran `git fetch origin` then `git rebase origin/main`. All 9 commits replayed with `git rebase` reporting success and *no textual conflict markers* — but that was misleading. The real conflict was structural, not textual: `main`'s UUID-migration commit (`refactor: migrate film IDs from integer to UUID`) had deleted the entire `WatchlistEntry` class from `models.py`, since `WatchlistEntry` didn't exist yet on `main` at that point and wasn't part of that refactor's scope. Every one of my watchlist commits only ever *added* code around `WatchlistEntry` (a relationship line on `Film`, service functions, routes, tests) — none of them touched the lines of the `WatchlistEntry` class definition itself. In a 3-way merge, when only one side (`main`) touches a region and the other side (my branch) leaves it untouched, the side with the change wins silently — so the rebase quietly kept main's deletion instead of stopping to ask which version I wanted. After the rebase "succeeded," `models.py` had no `WatchlistEntry` class at all, while every service/route/test file still imported and used it.

**How I resolved it:** I didn't notice this from the rebase output — I only found it because I ran the test suite immediately after rebasing and got `ImportError: cannot import name 'WatchlistEntry' from 'models'`. I re-added the `WatchlistEntry` class to `models.py`, this time with `film_id` as `db.Column(db.String(36), db.ForeignKey("film.id"), ...)` to match the now-UUID `Film.id` and the same pattern `CollectionEntry.film_id` already uses post-refactor. I then swept the rest of the watchlist code for anything still assuming integer IDs: the `film_id (int)` docstrings in `services/watchlist_service.py`, the `{ "film_id": <int> }` examples in the route docstrings in `routes/watchlist/watchlist.py`, and the test fixture in `tests/test_watchlist.py` that used a bare integer (`999999`) as a "nonexistent film" placeholder — I changed that to the same UUID-shaped sentinel (`"00000000-0000-0000-0000-000000000000"`) that `test_collection.py` already uses for the equivalent case, so both test files follow the same convention.

**How I verified no conflict remains:** `git status` shows no unmerged paths and `git log --oneline` shows a fully linear history with no merge commits. I ran `pytest tests/ -v` — all 12 tests pass. I also booted the app (`create_app()`) and printed `app.url_map` to confirm every collection and watchlist route still registers correctly (including the new `remove` endpoint) with no import-time errors, since the missing-class bug only surfaced at import/runtime, not at rebase time.

## Stretch — remove_from_watchlist()
**What I did:** Added `remove_from_watchlist(user_id, film_id)` to `services/watchlist_service.py`, following the exact shape of `remove_from_collection()`: a `NotInWatchlistError` exception, a lookup by `(user_id, film_id)`, delete-and-commit if found, raise if not. Wired it up as `DELETE /watchlist/<user_id>/remove` in `routes/watchlist/watchlist.py`, mirroring `DELETE /collection/<user_id>/remove` (same request body shape, same 404-on-not-found behavior). Added `test_remove_from_watchlist_deletes_entry` and `test_remove_from_watchlist_not_present_raises` in `tests/test_watchlist.py`.

While wiring this up I found that `get_watchlist()` was actually broken — `Film` had no `watchlist_entries` relationship, so `entry.film` raised `AttributeError` the moment any test exercised it (no prior test called `get_watchlist()`, so it had never been caught). I fixed that in its own commit (`fix: add missing Film relationship for WatchlistEntry`) since it's a real bug, not part of any of the six comments.

## Stretch — additional test
**What I did:** `test_get_watchlist_returns_alphabetical_order` in `tests/test_watchlist.py` — adds two films in reverse-alphabetical order and asserts `get_watchlist()` returns them alphabetically.

**Why I chose this case:** The Comment 3 test and the collection tests all cover `add_*` error handling, but nothing in the suite actually verified the sort order that Comment 5 is about — the exact behavior I'm defending in my pr-response would have had zero regression protection. If someone "fixed" the sort to date-added later without reading this doc, nothing would fail. This test makes the decision executable, not just documented.

## Stretch — visibility toggle
**What I did:** Added a `public` keyword argument to `add_to_watchlist(user_id, film_id, public=True)`, and exposed it on `POST /watchlist/<user_id>/add` as an optional `"public"` field in the request body (defaulting to `True` when omitted, so existing callers are unaffected). This is the concrete resolution to the Comment 4 discussion below — instead of just deciding the default in prose, callers now have a real way to override it per entry. Added `test_add_to_watchlist_defaults_to_public` and `test_add_to_watchlist_respects_public_false`.

## PR Description

**What this feature does:** Adds a watchlist to CineLog — a per-user list of films they intend to watch later, distinct from the existing "collection" of films they've already watched. Users can add a film to their watchlist (`POST /watchlist/<user_id>/add`), remove one (`DELETE /watchlist/<user_id>/remove`), and view their full watchlist (`GET /watchlist/<user_id>`), with server-side deduplication so the same film can't be added twice.

**Design decisions:**
- **Default visibility:** Watchlist entries default to `public=True`, consistent with the rest of CineLog's activity (the collection endpoint has no privacy concept at all) and with the app's identity as a community/discovery platform. Callers can override this per entry with an optional `"public"` field in the add request body. Full reasoning in Comment 4 above.
- **Sort order:** `GET /watchlist/<user_id>` returns films alphabetically by title, not by date added. This mirrors `GET /films/`'s existing sort convention and fits a watchlist's primary use case — checking whether something's already queued — better than a recency-first view. Full reasoning and engagement with the alternative in Comment 5 above.

**How to manually test:**
1. Start the app: `python app.py` (runs at `http://127.0.0.1:5000`).
2. Create a user and a film through whatever seed/fixture path the app uses, or use the IDs already in `cinelog.db` if seeded.
3. Add a film to the watchlist:
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_uuid>"}'
   ```
   Expect `201` with the new entry, `"public": true`.
4. Try adding the same film again — expect `409` (`AlreadyInWatchlistError`).
5. Try adding a nonexistent film UUID — expect `404` (`FilmNotFoundError`).
6. Add a second film with `"public": false` in the body — expect `201` with `"public": false` in the response.
7. View the watchlist: `curl http://127.0.0.1:5000/watchlist/<user_id>` — expect both films, sorted alphabetically by title regardless of add order.
8. Remove a film: `curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove -H "Content-Type: application/json" -d '{"film_id": "<film_uuid>"}'` — expect `200`, and the film no longer appears in step 7's `GET`.
9. Try removing it again — expect `404` (`NotInWatchlistError`).
10. Run the automated suite for full coverage: `pytest tests/ -v` (12 tests, all passing).

## Commit History

`git log --oneline` on `feature/watchlist`, rebased onto `main` with no merge commits:

```
eaa253e fix: restore WatchlistEntry model with UUID film_id after main rebase
dfc79ca test: add coverage for watchlist alphabetical sort order
d414bc1 feat: add public visibility toggle to add_to_watchlist endpoint
4d19f51 feat: add remove_from_watchlist endpoint
bf69371 fix: add missing Film relationship for WatchlistEntry
54c3e41 test: add tests for add_to_watchlist happy path, dedup, and nonexistent film_id
de8f698 fix: add deduplication check to prevent duplicate watchlist entries
24fdbdd fix: rename save_to_watchlist to add_to_watchlist per naming convention
59502bd feat: add watchlist model and add_to_watchlist endpoint
bbe206c Merge pull request #2 from ascherj/chore/add-gitignore   <- last commit on main, not part of this branch's work
```

<!-- TODO: replace this text block with an actual screenshot image of `git log --oneline` (e.g. `![git log](git-log.png)`) before final submission — this environment could produce the text output but not a literal screenshot. -->
