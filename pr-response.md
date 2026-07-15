# PR Response Doc — CineLog Watchlist Feature

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, and updated the import and call site in `routes/watchlist/watchlist.py`.
**How I verified:** Used VSCode's find-all-references for `save_to_watchlist` and `add_to_watchlist` across all files to confirm no references to the old name remained and both call sites (definition + route) used the new name.

## Comment 2 — Deduplication
**What I did:** Added dedup logic to `add_to_watchlist()` in `services/watchlist_service.py`, following the same pattern as `add_to_collection()` in `services/collection_service.py`: query for an existing `WatchlistEntry` by `user_id`/`film_id`, and raise a new `AlreadyInWatchlistError` if one is found, before creating the entry.
**How I verified:** Compared the new code side-by-side against `add_to_collection()` to confirm the same existence-check-then-raise structure and naming convention (`AlreadyInCollectionError` → `AlreadyInWatchlistError`).

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with the same `app`/`sample_user`/`sample_film` fixtures as `tests/test_collection.py`, and added `test_add_to_watchlist_nonexistent_film_raises`, mirroring `test_add_to_collection_nonexistent_film_raises`, asserts that calling `add_to_watchlist()` with a nonexistent `film_id` raises `FilmNotFoundError`.
**How I verified:** Ran `python -m pytest tests/test_watchlist.py -v` and got 1 passed.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for `WatchlistEntry`, and treat this as an intentional choice rather than an inherited default.

**Reasoning:** The watchlist and the collection serve different purposes: the collection is a personal record of films already watched (it has no `public` field at all), while the watchlist is forward-looking — films a user *wants* to watch. That makes it naturally social: a public-by-default watchlist lets friends see what you're planning to watch and suggest films, without every user having to discover and flip a per-entry visibility toggle first. If the default were private, the sharing/discovery behavior the field exists to support would rarely get used in practice, since most users don't proactively change settings they don't know exist — the feature would be dead on arrival for the majority of users. Defaulting to public optimizes for the list being immediately useful as a lightweight social signal from the moment a film is added.

**Tradeoff acknowledged:** The cost is privacy exposure for users who don't want their to-watch list visible (e.g., films tied to sensitive topics, or just not wanting to broadcast their queue) and who don't realize it's public until after the fact — new users especially won't think to check a setting they didn't set themselves. That's a real risk, and if we see evidence of it causing problems (support complaints, users manually hiding entries en masse), the right follow-up is a one-time onboarding prompt or an account-level default users can set once, rather than flipping the system default to private and undermining the sharing use case for everyone else.

## Comment 5 — Sort order
**My position:** I agree with the reviewer. The watchlist should default to date-added order (most recent first) instead of alphabetical.

**Reasoning:** Alphabetical order is arbitrary with respect to how someone actually uses a watchlist. It doesn't reflect intent, priority, or recency, just where a title happens to fall in the alphabet. Date-added order carries meaning because the most recently added film is usually the one a user just heard about or got excited over (from a trailer, a friend's recommendation, etc.), so surfacing it first matches what the user is most likely looking for when they open their watchlist. Alphabetical sorting only helps if someone is hunting for one specific known title, which is a search problem, not a default-ordering problem. This also brings the watchlist in line with `get_collection()`, which already sorts by `date_added.desc()`, so both features in the app follow the same "recent activity first" mental model instead of the watchlist being the odd one out.

**Engagement with reviewer's point:** The reviewer's comment, that most users want to see what they added recently, matches the actual use pattern of a "to-watch" list. It's a running, growing queue, not a fixed reference list a user browses by name. Alphabetical order actively works against that, burying a film added five minutes ago under whatever else starts with "A." I don't see a strong case for alphabetical as the default.

## Comment 6 — Rebase
**What conflicted:** Rebasing `feature/watchlist` onto `origin/main` pulled in main's UUID refactor (`Film.id` and `CollectionEntry.film_id` changed from `Integer` to `String(36)`), which touched the same region of `models.py` as my branch's addition of the `WatchlistEntry` class. The conflict resolution ended up taking main's version of `models.py` wholesale, which silently deleted the entire `WatchlistEntry` class, it wasn't a merge marker left in the file, the class was just gone after the rebase completed. This wasn't caught immediately because it's a valid Python file either way; it only surfaced later as an `ImportError: cannot import name 'WatchlistEntry' from 'models'` when running the tests.

**How I resolved it:** Re-added the `WatchlistEntry` class to `models.py`, using the version from the pre-rebase branch tip as the base. I also had to update `film_id` from `db.Integer` to `db.String(36)` to match the UUID refactor that main introduced — otherwise the foreign key to `Film.id` would have been left pointing at the old, pre-refactor integer ID scheme.

**How I verified no conflict remains:** Ran the full tests with `python -m pytest tests/ -v` and all 5 tests pass (4 from `test_collection.py`, 1 from `test_watchlist.py`), confirming `WatchlistEntry` imports correctly and its foreign key types are consistent with the rest of the post-refactor schema.

## PR Description

### What this feature does
Adds a watchlist to CineLog — a list of films a user wants to watch, separate from their collection of films already watched. Two endpoints:

- `GET /watchlist/<user_id>` — returns all films on a user's watchlist, with `date_added` and `public` metadata attached to each film.
- `POST /watchlist/<user_id>/add` — adds a film (`{ "film_id": "<uuid>" }`) to a user's watchlist.

Backed by a new `WatchlistEntry` model (`user_id`, `film_id`, `date_added`, `public`) and `services/watchlist_service.py` (`add_to_watchlist()`, `get_watchlist()`).

### Design decisions
- **Naming:** `add_to_watchlist()` (not `save_to_watchlist()`), matching the `add_to_*` convention already used by `add_to_collection()`.
- **Deduplication:** adding a film already on a user's watchlist raises `AlreadyInWatchlistError` instead of silently creating a duplicate entry, following the same existence-check pattern as `add_to_collection()`.
- **Default visibility (`public=True`):** intentional, not inherited — see Comment 4 for full reasoning. Watchlists are treated as a social/discovery list (unlike the collection, which has no visibility concept at all), so defaulting to public makes the sharing behavior usable without requiring users to find and flip a setting.
- **Sort order:** `get_watchlist()` currently sorts alphabetically by film title (`Film.title.asc()`). Per Comment 5, I agreed with the reviewer that it should default to date-added order (most recent first) instead — **this change is agreed on but not yet implemented in this PR**; it's a follow-up.

### Manual testing steps
There's no user/film-creation endpoint (films are expected to be seeded, users aren't created via the API yet), so testing goes through a Flask shell to set up data, then `curl` against the running app.

1. Start the app: `python app.py` (runs on `http://127.0.0.1:5000`).
2. In a separate terminal, open a shell in app context to create test data:
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
3. **Add a film to the watchlist:**
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
   Expect a `201` with the new entry (including `public: true`).
4. **View the watchlist:**
   ```
   curl http://127.0.0.1:5000/watchlist/<user_id>
   ```
   Expect a list containing the film just added, with `date_added` and `public` fields.
5. **Test deduplication:** repeat step 3 with the same `film_id`. Expect this to fail rather than create a second entry (currently unhandled at the route level — see the note on Comment 2's route gap — so this will surface as a 500 until that's addressed, not a clean error response).
6. **Test nonexistent film:** repeat step 3 with a made-up UUID for `film_id`. Expect a failure rather than a silently-created entry (same caveat — currently unhandled at the route level, so this surfaces as a 500 rather than a 404).
7. **Automated coverage:** `python -m pytest tests/ -v` — covers the nonexistent-film case at the service layer (`test_add_to_watchlist_nonexistent_film_raises`) plus the existing collection tests.

## AI Usage
For Comment 5 (sort order), after drafting my position that watchlists should default to date-added order, I asked Claude: "What counterargument would a careful code reviewer raise against this position? What tradeoff am I not acknowledging?"

Claude's response raised a tradeoff I hadn't considered: date-added order buries older watchlist entries at the bottom of the list, so a user could lose track of a film they added a while ago but still want to watch, which is arguably a worse failure mode for a watchlist than alphabetical order, which treats every entry equally regardless of age.

My final reasoning in Comment 5 does not incorporate this tradeoff. I decided my original argument still outweighed it, and kept my original reasoning.