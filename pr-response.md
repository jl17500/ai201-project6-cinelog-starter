# PR Response Doc — CineLog Watchlist Feature

## AI Usage

Used Claude 3 ways:

- Summarized and explained `models.py`, `collection_service.py`, `test_collection.py` before touched code or read review comments

- Rebase Bugging: Rebase reported "Successfully rebased," no conflicts. Claude reproduced it in sandbox, found `WatchlistEntry` dropped from `models.py`. Confirmed with `pytest` (`ImportError`). Full finding in Comment 6

- Verifying facts: My Comment 5 drafts suggested sorting by `WatchlistEntry.id` as an insertion proxy since I thought there was no timestamp column. Claude examined the schema: `id` is a UUID string (sorting by it isn't chronological), and `date_added` already exists. Rewritten using actual schema. Claude's part was fact-checking and pushback, not creating the argument; the positions and logic are mine.


---
 
## Comment 1 — Rename
 
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`. Updated import and call site in `routes/watchlist/watchlist.py`.
 
**How I verified:** Project wide search for `save_to_watchlist` returned zero matches. Full test suite passed after change.
 
---
 
## Comment 2 — Deduplication
 
**What I did:** Added `AlreadyInWatchlistError` exception. Added a check in `add_to_watchlist()` that queries for an existing `WatchlistEntry` (by `user_id` + `film_id`) and raises before creating a duplicate.
 
**How I verified:** Mirrored the exact pattern in `add_to_collection()` (`collection_service.py`), existence check, then dedup check, then create. Test suite passed after change
 
---
 
## Comment 3 — Missing test
 
**What I did:** Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, modeled directly on `test_add_to_collection_nonexistent_film_raises` in `test_collection.py`. Same fixture structure (`app`, `sample_user`), same assertion pattern (`pytest.raises`)
 
**How I verified:** Test passes standalone and as part of the full suite (5/5)
 
---
 
## Comment 4 — Default visibility
 
**My position:** Keep `public=True` as the default.
 
**Reasoning:** `CollectionEntry` already defaults to public. Splitting visibility by list type (public collections, private watchlists) forces users to remember which one auto-shares so a consistent default avoids that.
 
**Tradeoff acknowledged:** Exposes users who want a private "watch later" list by default. Easier fix: prompt users to opt out (`public=False`) than to retrofit discovery onto a platform where lists start hidden.
 
---
 
## Comment 5 — Sort order
 
**My position:** Default to `date_added` descending (most recent first). No additional sort option.
 
**Reasoning:** A watchlist is an active queue, not a static catalog. Users open it to pick something to watch now, a film added moments ago is top-of-mind. Alphabetical buries it under whatever starts with A–V.
 
**Engagement with reviewer's point:** Agree with `@dev-lead`. Considered adding a `?sort=alphabetical` param on top of the default, but cut it as unrequested API surface for a case the maintainer didn't raise, a clear default is enough.
 
**Tradeoff acknowledged:** Chronological order makes scanning for duplicates harder as a list grows. Mitigated by Comment 2, dedup already happens at the API level, so there's less need to scan manually.
 
---
 
## Comment 6 — Rebase
 
**What conflicted:** `.gitignore` had a trivial add/add conflict, both branches added one independently. Merged into one clean list.
 
Bigger issue: rebasing onto `main`'s UUID refactor caused `WatchlistEntry` to be silently **dropped** from `models.py`, no conflict markers, "Successfully rebased" printed clean. The two branches' patches applied to non-overlapping lines, so git merged them into something broken without flagging it.
 
**How I resolved it:** Ran `pytest` right after the rebase, got `ImportError: cannot import name 'WatchlistEntry'`. Readded the class to `models.py` with `film_id` as `db.String(36)` (was `db.Integer`) to match the UUID refactor. Fixed a stale docstring in `watchlist_service.py` still describing `film_id` as integer.
 
**How I verified no conflict remains:** `pytest tests/ -v` passes 5/5. `git log --oneline` confirms a linear history, no merge commits between `feature/watchlist` and `main`.
 

---

## Commit History

```
6403385 docs: add pr-response.md with visibility and sort order decisions
c3115c7 fix: restore WatchlistEntry model with UUID film_id after main rebase
67f7270 fix: default watchlist sort to date added per maintainer review
f48eb68 test: add test for nonexistent film_id in add_to_watchlist
4d68189 fix: add deduplication check to prevent duplicate watchlist entries
b94d1cd fix: rename save_to_watchlist to add_to_watchlist per naming convention
9b9f488 chore: add .gitignore for files
994e7e9 fix: update film retrieval method to use db.session.get in collection and watchlist services
8f6c84b feat: add WatchlistEntry model and initial watchlist service/endpoints
```

Nine standard format commits, one logical change per commit, and no merging commits are made before the main. One inherited commit was rewritten using interactive rebase: "added watchlist model and endpoint fixed a bug more changes" = `feat: add WatchlistEntry model and basic watchlist service/endpoints`; everything else was already clean.


---

## PR Description

Adds a watchlist tool to CineLog. Users may store movies to watch later, keeping them distinct from their library of previously watched movies. Includes a new `WatchlistEntry` database model, two service functions (`add_to_watchlist()`, `get_watchlist()`), and two endpoints: `GET /watchlist/<user_id>` and `POST /watchlist/<user_id>/add`.


**Design decisions:**

- **Default visibility:** New watchlists are made public by default. Collections are already set to public, thus this ensures uniform sharing behavior across both list types rather than confusing users with conflicting settings on the same account.

- **Sort order:** Watchlists are not alphabetical; instead, they display the most recent additions first. The movie they just added should be simple to locate, not buried behind something they saved weeks ago, as people use watchlists to pick what to watch tonight.

**Manual testing steps:**
1. Start the app: `python app.py`
2. Open a Python shell, create one user and one film, note their UUIDs as the app has no signup form or "add film" endpoint, so test data has to be created directly.
3. Add the film to the watchlist: `POST /watchlist/<user_id>/add` with body `{"film_id": "<film_id>"}`. Should return `201` with the new entry.
4. View the watchlist: `GET /watchlist/<user_id>`. Should list that film.
5. Try adding the same film again. Should be rejected, not duplicated.
6. Try adding a film ID that doesn't exist. Should return a clean error, not a server crash.