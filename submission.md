# Mixtape Bug Hunt Submission

## AI Usage

I used ChatGPT to help explain unfamiliar code, understand the project structure, trace execution paths, and better understand Python and Flask debugging steps. I used it to help identify where each reported issue was likely located based on the code organization, explain what individual functions were doing, and compare similar code paths to find inconsistencies. I verified all suggested fixes by reading the relevant code myself, reproducing each issue through the API, restarting the Flask server when necessary, and running the project's test suite after each fix to ensure no existing functionality was broken.

---

# Codebase Map

## app.py

Creates and configures the Flask application. It initializes the database and registers the route blueprints for feed, playlists, songs, and users.

## models.py

Defines the SQLAlchemy database models and association tables used throughout the application, including users, songs, playlists, playlist entries, listening events, ratings, notifications, friendships, and tags.

## routes/

Contains all API route handlers. Each route receives HTTP requests, validates inputs, and delegates business logic to the appropriate service.

- `routes/feed.py` – Handles activity feed and "Listening Now" endpoints.
- `routes/playlists.py` – Handles playlist creation, retrieving playlists, and adding songs.
- `routes/songs.py` – Handles song lookup, search, listening events, and song ratings.
- `routes/users.py` – Handles user information, listening streaks, notifications, and marking notifications as read.

## services/

Contains nearly all business logic used by the application.

- `services/feed_service.py` – Builds the activity feed and listening-now results.
- `services/playlist_service.py` – Handles playlist creation and retrieving playlist songs.
- `services/search_service.py` – Performs song search operations.
- `services/notification_service.py` – Creates and retrieves notifications.
- `services/streak_service.py` – Calculates and updates listening streaks.

## seed_data.py

Recreates the database and populates it with sample users, friendships, playlists, songs, tags, ratings, listening history, and notifications used for testing.

---

# Data Flow Example

When a user requests all songs in a playlist:

1. The client sends a GET request to `/playlists/<playlist_id>/songs`.
2. `routes/playlists.py` receives the request.
3. The route calls `get_playlist_songs()` inside `services/playlist_service.py`.
4. The service verifies the playlist exists.
5. The playlist entries are queried and joined with the Songs table.
6. Songs are ordered by playlist position.
7. Each song is converted into a dictionary and returned as JSON to the client.

---

# Issue #5 — The last song in a playlist never shows up

## How I reproduced it

I requested:

`GET /playlists/0dad0216-1718-4d0a-bf40-e0883d2def7d/songs`

for the "Friday Energy" playlist. The playlist should contain 7 seeded songs, but the response only returned `"count": 6`.

## How I found the root cause

I traced the endpoint from `routes/playlists.py` into `services/playlist_service.py`. The query correctly returned every song ordered by playlist position, but the return statement sliced the results before converting them into dictionaries.

## The root cause

The function returned:

```python
songs[:-1]
```

instead of the complete list. Since Python slices exclude the last element, the newest song in every playlist was always omitted.

## My fix and side-effect check

I removed the slice so the function returns every song in the list.

After restarting Flask, the same endpoint correctly returned `"count": 7`, confirming that all playlist songs are now included.

---

# Issue #3 — Duplicate search results

## How I reproduced it

I searched using:

`GET /songs/search?q=Anthem`

The same song appeared multiple times in the search results even though only one song matched the query.

## How I found the root cause

I followed the request from `routes/songs.py` into `services/search_service.py`. The search query unnecessarily joined the `song_tags` table before filtering by title and artist.

## The root cause

Songs with multiple tag records produced multiple database rows during the join. Since the query did not remove duplicates before returning the results, identical songs appeared multiple times.

## My fix and side-effect check

I removed the unnecessary join and added `.distinct()` to ensure only unique songs are returned.

I verified the fix by searching for "Anthem" again, which now returned a single copy of the matching song. I also ran `pytest tests/test_search.py`, and all search tests passed.

---

# Issue #4 — Missing rating notifications

## How I reproduced it

I submitted a rating using:

`POST /songs/<song_id>/rate`

The rating was saved successfully, but no notification appeared when checking the original song owner's notifications.

## How I found the root cause

I compared the rating workflow in `routes/songs.py` and `services/notification_service.py` with the playlist notification workflow. The playlist function created notifications after completing its action, while the rating function only saved the rating.

## The root cause

The `rate_song()` function never called `create_notification()` after storing the rating. As a result, rating notifications were never created even though the rating itself was successfully saved.

## My fix and side-effect check

After committing the rating to the database, I added a call to `create_notification()` whenever someone rates another user's shared song.

I verified the fix by rating a song from another user and confirming that a new notification appeared in that user's notification list. All project tests also continued to pass.

---

# Issue #1 — My listening streak keeps resetting

## How I reproduced it

I ran the provided streak tests. The test that simulated listening on Saturday followed by Sunday failed because the streak reset to 1 instead of increasing to 2.

## How I found the root cause

I examined `services/streak_service.py` and followed the streak update logic inside `update_listening_streak()`. The failure occurred during the condition that handles consecutive listening days.

## The root cause

The code checked:

```python
today.weekday() != 6
```

before incrementing the streak. Since Python uses `weekday() == 6` for Sunday, listening on Sunday incorrectly skipped the increment logic and reset the streak instead.

## My fix and side-effect check

I removed the unnecessary weekday check and allowed any consecutive day (`days_since_last == 1`) to increment the streak.

After making the change, all streak tests passed, including the Sunday test. Running the full test suite confirmed all 13 tests passed successfully.