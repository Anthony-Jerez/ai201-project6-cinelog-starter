# PR Response Doc — CineLog Watchlist Feature

## AI Usage

First, codebase orientation before writing anything. Before renaming `save_to_watchlist`, I had it grep the repo for every call site so I knew the rename was safe. Before writing the dedup check, I had it walk through `add_to_collection()`'s existing pattern in services/collection_service.py so the watchlist version followed the same convention instead of inventing something new.

Second, diagnosing the rebase. Before running `git rebase origin/main`, I had it check with `git merge-tree` whether the main branch's UUID refactor would actually produce a textual conflict. It came back clean, no conflict markers, which is exactly why the rebase completed without stopping, and why the real problem (main had deleted the `WatchlistEntry` class) only showed up after running the test suite, not from resolving `<<<<<<<` markers.

Third, verifying the final commit history against the conventional commits spec, and helping tighten the prose in this doc.

For Comments 4 and 5, I wrote my own position and reasoning first, the arguments above are mine. I asked Claude to write that reasoning up in this doc's format, not to come up with the argument itself. The core claims (public default lowers sharing friction and matches how similar apps behave, date added sort should match `get_collection()` for consistency) are mine.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in services/watchlist_service.py, then updated the one call site in routes/watchlist/watchlist.py (both the import line and the actual function call in `add_film`).

**How I verified:** Ran a repo wide grep for `save_to_watchlist` before and after the change. Before, it turned up 3 hits: the function definition and the two references in the route file. After the rename, the grep comes back empty, so nothing was missed.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a dedup check inside `add_to_watchlist()`. I copied the pattern from `add_to_collection()` in services/collection_service.py: check the film exists first (raise `FilmNotFoundError` if not), then query for an existing `WatchlistEntry` with the same `user_id` and `film_id`, and raise if one is found, before ever constructing or committing the new entry. Same order of checks, same style of exception, just scoped to the watchlist model.

I noticed `WatchlistEntry` doesn't have a `UniqueConstraint` like `CollectionEntry` does, so there's no DB level backstop here, just the application level check. That's out of scope for this comment but worth flagging for a follow up.

**How I verified:** Ran the full test suite after the change to confirm nothing regressed, then traced through the logic by hand: calling `add_to_watchlist()` twice with the same user and film hits the `existing` query on the second call and raises before any insert happens.

## Comment 3 — Missing test
**What I did:** Added tests/test_watchlist.py with `test_add_to_watchlist_nonexistent_film_raises`, modeled directly on `test_add_to_collection_nonexistent_film_raises` in tests/test_collection.py. Same `app` fixture (in memory SQLite via `create_app`), same `sample_user` fixture, same fake UUID string as the film id, same `pytest.raises(FilmNotFoundError)` assertion.

**How I verified:** Ran `pytest tests/test_watchlist.py -v`, passed. Then ran `pytest tests/ -v` for the full suite (5 tests total now), all passing.

## Comment 4 — Default visibility
**My position:** Keeping `public=True` as the default for `WatchlistEntry`.

**Reasoning:** CineLog is a community film tracking app, not a personal notes tool. People who sign up expect to share taste and see what others are planning to watch, and a watchlist's main social value comes from being visible. Letterboxd and Trakt both default watchlists to public for the same reason: if a user has to flip a toggle to be seen, most won't bother, and the platform loses that signal. Defaulting to public means the common case (share what I'm planning to watch) takes no extra effort, and the less common case (hide it) takes one deliberate action.

**Tradeoff acknowledged:** The counterargument is real. Some users will feel exposed queuing up a film they never end up watching, or adding something they'd rather not have public. A private by default model lowers that risk. My response is that the fix for this is a clear per entry visibility toggle in the UI, which the `public` field already supports, not flipping the default against the platform's own social purpose. Users who want privacy get an explicit path. The default should serve the majority case.

## Comment 5 — Sort order
**My position:** Implemented date added descending (most recent first) in `get_watchlist()`, matching what `get_collection()` already does.

**Reasoning:** A watchlist is meant to be acted on, not just referenced. When someone opens it, they're usually deciding what to watch next, and that's most often something they added recently (last night's trailer, a friend's recommendation this week). Alphabetical sort is built for lookup, finding a specific title you already know is on the list. Date added sort is built for action, surfacing what's fresh. For a watchlist, recency is the more useful default.

**Engagement with reviewer's point:** You made a fair point that alphabetical feels intuitive for reference style lists, the way a bookshelf or queue works. I don't think that's wrong in the abstract. But `get_collection()` already sorts by date added descending, and that list is the more permanent, reference like one of the two. If the watchlist used alphabetical while the collection used date added, users would have to remember which list sorts which way. Matching the two keeps one consistent mental model across the app: everything is newest first. That consistency outweighs alphabetical's abstract intuitiveness here.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin` and checked what changed on main first. The only relevant commit was `refactor: migrate film IDs from integer to UUID`, which changed `Film.id` and `CollectionEntry.film_id` from `Integer` to `String(36)`. That commit was written against an older version of `models.py` that predated the watchlist model, so as part of the refactor it dropped the `WatchlistEntry` class entirely.

My branch never touched `models.py` in any of its commits (confirmed with `git show <commit> -- models.py` on each one), so when I ran `git rebase origin/main` it completed with zero conflict markers, no `<<<<<<<` to resolve by hand. But that clean rebase left `models.py` with no `WatchlistEntry` class at all, while `services/watchlist_service.py`, `routes/watchlist/watchlist.py`, and `tests/test_watchlist.py` still imported it. Running the test suite right after the rebase confirmed this with `ImportError: cannot import name 'WatchlistEntry' from 'models'`. So the real conflict was semantic, not textual, and git alone wouldn't have caught it.

Separately, my local `.gitignore` was untracked, and main had since added its own tracked `.gitignore` with the same content plus one extra line (`.pytest_cache/`). I removed my local copy before rebasing since main's version fully supersedes it, otherwise the rebase would have failed with an "untracked working tree files would be overwritten" error.

**How I resolved it:** Re-added the `WatchlistEntry` class to `models.py`, with `film_id` now `db.String(36)` pointing at the now UUID `film.id`, matching how `CollectionEntry.film_id` was migrated. While restoring it, I also noticed `get_watchlist()` calls `entry.film.to_dict()`, but there was never a declared relationship between `WatchlistEntry` and `Film`, only `CollectionEntry` has one (`Film.collection_entries = db.relationship("CollectionEntry", backref="film", ...)`). That looks like a bug that predates this rebase since nothing ever tested `get_watchlist()`. Since I was already rebuilding this class from scratch, I added the matching relationship (`Film.watchlist_entries = db.relationship("WatchlistEntry", backref="film", ...)`) so `entry.film` actually resolves. I also updated a stale docstring in `watchlist_service.py` that still said `film_id (int)` from before the refactor.

**How I verified no conflict remains:** Ran `pytest tests/ -v`, all 5 tests pass. Since no existing test exercises `get_watchlist()`, I also wrote a quick standalone script that creates a user and two films, adds them to a watchlist at different timestamps, and calls `get_watchlist()` directly, confirming it returns newest first and that `entry.film` resolves without raising. Finally, checked `git log --oneline --graph` and confirmed every commit on this branch has exactly one parent (checked with `git cat-file -p` on each), the only merge commit in the log (`Merge pull request #2 from ascherj/chore/add-gitignore`) belongs to main's own history from before I rebased, not something introduced by me.

## PR Description

### What this adds
A watchlist feature. Users can save films they want to watch later, separate from their collection of films already watched. Two endpoints:

- `GET /watchlist/<user_id>` returns the user's watchlist, newest added first.
- `POST /watchlist/<user_id>/add` with body `{"film_id": "<uuid>"}` adds a film to the user's watchlist. It rejects a `film_id` that doesn't exist and rejects a film that's already on the list.

### Design decisions
- **Default visibility**: new watchlist entries default to `public=True`. Full reasoning in Comment 4. Short version: CineLog is a social app, and low friction sharing is the more useful default than opt in visibility.
- **Sort order**: `get_watchlist()` returns entries newest added first (`date_added` descending), matching `get_collection()`. Full reasoning in Comment 5. Short version: consistency with the collection list, and optimizing for "what did I just add" over alphabetical lookup.

### Manual testing
There's no endpoint to create a `User` or `Film` (`routes/films.py` is read only against seeded data), so testing needs a user and film created directly through the app context first.

1. Start the app. Note: running `python app.py` as is triggers Flask's debug reloader, which crashes the first request with a `RuntimeError` about the SQLAlchemy instance not being registered. That's a pre-existing bug in `app.py`'s `__main__` block, unrelated to this feature, so start it without the reloader instead:
```
source .venv/bin/activate
python -c "from app import create_app; create_app().run(debug=False)"
```
2. In a separate shell, seed a test user and film:
```
python -c "
from app import create_app, db
from models import User, Film

app = create_app()
with app.app_context():
    user = User(username='testuser', email='test@example.com')
    film = Film(title='Paddington 2', year=2017, genre='Comedy')
    db.session.add_all([user, film])
    db.session.commit()
    print('user_id:', user.id)
    print('film_id:', film.id)
"
```
3. Add the film to the watchlist (swap in the ids printed above):
```
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_id>"}'
```
Expect a `201` with the new entry, `"public": true`.

4. View the watchlist:
```
curl http://127.0.0.1:5000/watchlist/<user_id>
```
Expect the film back with `date_added` and `public: true`.

5. Try adding the same film again. This should be rejected. Right now it surfaces as a `500` instead of a clean `409`, since the route doesn't catch `AlreadyInWatchlistError` yet. That gap is called out as a follow up in Comment 2, not something this PR fixes.

6. Add a second film and repeat step 4. The second film should now be first in the list, confirming the newest added first ordering from Comment 5.

### Final commit history

<img width="991" height="182" alt="Screenshot 2026-07-13 at 3 02 49 AM" src="https://github.com/user-attachments/assets/8a6a0b04-255c-45df-8f0d-48276ab56077" />

`git log --oneline` against `main`, after the interactive rebase into conventional commit format:
```
docs: add pr-response.md covering all six review comments and PR description
fix: update WatchlistEntry film_id to UUID after main branch refactor
fix: sort watchlist by date added descending instead of alphabetical
test: add test for nonexistent film_id in add_to_watchlist
fix: add deduplication check to prevent duplicate watchlist entries
fix: rename save_to_watchlist to add_to_watchlist per naming convention
fix: update film retrieval method to use db.session.get in collection and watchlist services
feat: add watchlist model and add_to_watchlist endpoint
```
8 commits, all conventional, no merge commits in this branch's own history (the branch is rebased onto `origin/main`, not merged).
