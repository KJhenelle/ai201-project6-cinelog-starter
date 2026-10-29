# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:**
updated all instances of save_to_watchlist to add_to_watchlist
**How I verified:**
i used ctrl+f to find all instances of save_to_watchlist replaced them with  add_to_watchlist then asked claude to verify there were no missed lines

## Comment 2 — Deduplication
**What I did:**
Added a duplicate check to `add_to_watchlist()` in `services/watchlist_service.py`, following the pattern `add_to_collection()` uses in `services/collection_service.py`.

The pattern in `add_to_collection()`:
- **What the check looks for:** after confirming the film exists, it queries `CollectionEntry.query.filter_by(user_id=user_id, film_id=film_id).first()`. It looks for an existing row with the same user and the same film, so the same film can be in different users' collections but only once in one user's.
- **What happens on a duplicate:** it does not insert anything and does not quietly return the existing entry. It raises `AlreadyInCollectionError` with the message "Film '<id>' is already in this user's collection". The caller has to handle the error.

What I changed to match it:
- Added `AlreadyInWatchlistError`, the watchlist version of `AlreadyInCollectionError`.
- After the film-exists check, `add_to_watchlist()` runs the same query against `WatchlistEntry` (`filter_by(user_id=user_id, film_id=film_id).first()`) and raises `AlreadyInWatchlistError` if a row comes back. The order is the same as in `add_to_collection()`: film exists, then duplicate, then insert.
- Added the new error to the docstring's Raises section.

Where it differs from the collection side: `CollectionEntry` also has a database-level `UniqueConstraint("user_id", "film_id")` as a backstop. `WatchlistEntry` doesn't have one yet, so for now the duplicate protection is the service-level check only. Two simultaneous requests could still both pass it. I plan to add the constraint to `models.py`.

Not done yet: `routes/watchlist/watchlist.py` doesn't catch `AlreadyInWatchlistError`, so a duplicate POST would currently return a 500 instead of a 409.
**How I verified:**
I compared the new check line by line against `add_to_collection()` and ran the full suite (`pytest tests/ -v`, 5 passed) after the change. That suite doesn't exercise the duplicate path for the watchlist yet: the only watchlist test is the nonexistent-film one from Comment 3. So the duplicate check itself is untested, and a duplicate-raises test modeled on `test_add_to_collection_duplicate_raises` is still to do.

## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`. I modeled it on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`, using the same structure:
- The same `app` fixture (in-memory SQLite, `create_all` / `drop_all`) and the same `sample_user` fixture.
- A docstring that states the expected behavior, and a `# ── Section ──` banner.
- A fake film ID, `"00000000-0000-0000-0000-000000000000"`, passed inside `with pytest.raises(FilmNotFoundError)`.

I left out the `sample_film` fixture because this test doesn't need a film. I used the UUID-style fake ID, as the collection test does, so the test still works after the UUID rebase in Comment 6. The test imports `FilmNotFoundError` from `services.collection_service`, because `watchlist_service` reuses that exception instead of defining its own.
**How I verified:**
Ran `pytest tests/test_watchlist.py -v` (1 passed), then `pytest tests/ -v` (5 passed, the 4 collection tests plus this one). I haven't confirmed that the test fails when the film check is removed.

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

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->