# PR Response Doc — CineLog Watchlist Feature

## AI Usage

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` convention (e.g. `add_to_collection`). Updated the import and call site in `routes/watchlist/watchlist.py`.
**How I verified:** Ran `grep -rn "save_to_watchlist" --include="*.py" .` to confirm no references remained, then ran `pytest tests/ -v` — all 4 tests passed.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a dedup check in `add_to_watchlist()` that queries for an existing `WatchlistEntry` by `user_id`/`film_id` before inserting, mirroring the pattern in `add_to_collection()`.
**How I verified:** Ran `pytest tests/ -v` — all 4 existing tests still pass.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, mirroring `test_add_to_collection_nonexistent_film_raises` from `test_collection.py` — same fixture setup (`app`, `sample_user`, `sample_film`), same fake-UUID pattern, asserts `FilmNotFoundError` is raised.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` (passed), then `pytest tests/ -v` — all 5 tests pass.

## Comment 4 — Default visibility
**My position:** `public=True` should stay the default.
**Reasoning:** CineLog is built to be social — people already share collections and ratings, and see what other users are watching. A watchlist is just another part of that. It's similar to Goodreads, where people can see what books you want to read. Keeping it public by default helps people discover more movies through what others want to watch.
**Tradeoff acknowledged:** A watchlist usually isn't something people put a lot of thought into — sometimes it's just random movies someone wants to watch later, so not everyone may want that visible. That's the strongest reason to default to `public=False`. But I still think `public=True` is better: if there's an easy visibility toggle, users who want privacy can switch it. If the default were private, most people would never change the setting, and the social side of the feature would barely get used.

## Comment 5 — Sort order
**My position:** Agreed with the maintainer — changed sort order from alphabetical (`Film.title.asc()`) to date-added descending (`WatchlistEntry.date_added.desc()`).
**Reasoning:** A watchlist is meant to reflect what a user wants to watch *right now*, and the most recently added film is usually the most relevant to them. This also matches the existing pattern in `get_collection()`, which already sorts by `date_added.desc()` — keeping sort behavior consistent across collection and watchlist features.
**Engagement with reviewer's point:** The maintainer's reasoning was that most users want to see what they added recently rather than an alphabetical list, which matches how people actually use a watchlist — as a running, evolving list rather than a static reference. Alphabetical sorting made sense for browsing a fixed catalog, but a watchlist changes constantly, so recency is the more useful ordering signal.

## Comment 6 — Rebase
**What conflicted:** Rebasing feature/watchlist onto upstream/main produced a conflict on .gitignore (both branches added one, with slightly different contents). More importantly, the automatic rebase merge silently dropped the WatchlistEntry model class from models.py without flagging a conflict, since it landed in the same region of the file that main's UUID refactor touched.
**How I resolved it:** Merged both versions of .gitignore by hand and continued the rebase. Afterward, tests/test_watchlist.py failed to import WatchlistEntry. I traced it using git show on the pre-rebase commit to confirm the class existed before the rebase, then manually re-added WatchlistEntry to models.py with film_id as db.String(36) (UUID) instead of db.Integer, matching the refactored Film and CollectionEntry models. Also updated a stale int type comment in routes/watchlist/watchlist.py to reflect the UUID string type.
**How I verified no conflict remains:** Ran git status to confirm a clean rebase with no unmerged paths, and pytest tests/ -v — all 5 tests pass after restoring the model.

## PR Description
