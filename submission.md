# Mixtape Bug Hunt Submission

## Milestone 1: Codebase Map

### Application Structure

Mixtape is a Flask-based social music application that allows users to share and search for songs, rate songs, create collaborative playlists, track listening streaks, view friends' listening activity, and receive notifications. The application follows a route-service-model structure in which routes handle HTTP requests and responses, services contain most of the business logic, and SQLAlchemy models define and connect the application's database entities.

### Main Files and Responsibilities

#### `app.py`

`app.py` implements the Flask application factory and database setup. The `create_app()` function configures the SQLite database and secret key, initializes SQLAlchemy, and registers the song, playlist, user, and feed Blueprints with their corresponding URL prefixes. It also creates the database tables within the application context before returning the configured Flask application.

#### `models.py`

`models.py` defines seven SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, and `Notification`. It also defines association tables for user friendships, song tags, and playlist entries. The `playlist_entries` table stores the position of each song in a playlist, the user who added it, and the time it was added. Relationships connect users to shared songs, ratings, listening events, notifications, playlists, and friends.

#### `routes/songs.py`

`routes/songs.py` defines endpoints for searching songs, retrieving individual songs, rating songs, and recording listening events. The routes validate request data and delegate business logic to `search_service.py`, `notification_service.py`, and `streak_service.py`.

#### `routes/playlists.py`

`routes/playlists.py` defines endpoints for creating playlists, retrieving playlist details, retrieving playlist songs, and adding songs to playlists. It validates request data and delegates business logic to `playlist_service.py` and `notification_service.py`.

#### `routes/users.py`

`routes/users.py` defines endpoints for retrieving user information, checking listening streaks, retrieving notifications, and marking notifications as read. User retrieval accesses the `User` model directly, while streak and notification operations are delegated to the appropriate service functions.

#### `routes/feed.py`

`routes/feed.py` defines endpoints for retrieving a user's friends-listening-now feed and activity feed. It delegates the business logic to `feed_service.py` and formats the returned results as JSON responses.

#### `services/streak_service.py`

`streak_service.py` contains the business logic for recording listening events and maintaining user listening streaks. It creates `ListeningEvent` records, updates a user's streak based on the date of the previous listening event, and retrieves the current streak value.

#### `services/feed_service.py`

`feed_service.py` contains the business logic for the friends-listening-now and activity feeds. It queries listening events from a user's friends, orders events by recency, retrieves the associated user and song records, and formats the results. The listening-now feature also applies a time threshold and returns only the most recent event for each friend.

#### `services/search_service.py`

`search_service.py` contains the business logic for searching and retrieving songs. The `search_songs()` function performs a case-insensitive search against song titles and artist names and joins the `song_tags` association table. The `get_song()` function retrieves a specific song by its ID.

#### `services/notification_service.py`

`notification_service.py` handles notification creation and retrieval, marking notifications as read, adding songs to playlists, and rating songs. It performs database queries, applies business rules, creates or updates records, and commits database changes.

#### `services/playlist_service.py`

`playlist_service.py` contains the business logic for creating and retrieving playlists. It validates playlist creators, creates new playlist records, retrieves playlist metadata, retrieves playlists created by a user, and queries playlist songs through the `playlist_entries` association table in ascending position order.

### Data Flow: Rating a Song

When a user rates a song, the client sends a `POST` request to `/songs/<song_id>/rate`. The `rate()` function in `routes/songs.py` reads the `user_id` and `score` from the JSON request and validates that both values were provided. The route then calls `rate_song()` in `services/notification_service.py`.

The `rate_song()` service function validates that the score is between 1 and 5, retrieves the corresponding `Song` and `User` records, and checks whether the user has previously rated the song. If a rating already exists, its score is updated. Otherwise, a new `Rating` record is created. The changes are committed to the database, and the `Rating` object is returned to the route. Finally, `routes/songs.py` converts the rating to a dictionary and returns it as a JSON response with HTTP status 201.

### Pattern I Noticed

The main architectural pattern is separation between routes, services, and models. Routes primarily handle request parsing, input validation, error handling, and response formatting. Service functions contain most of the business logic and database operations, while models define the database entities and their relationships. A feature can therefore be traced from an HTTP endpoint in a route file, through a service function, to the SQLAlchemy models and database.

### Initial Bug-Fix Plan

After reviewing the five open issues and their affected service files, I plan to investigate the following three bugs first:

1. **Issue #5: The last song in a playlist never shows up**  
   I will trace the playlist retrieval flow from `routes/playlists.py` to `playlist_service.get_playlist_songs()` and examine how the ordered query results are converted into the returned song list.

2. **Issue #1: My listening streak keeps resetting**  
   I will trace the listening flow from `routes/songs.py` to `streak_service.record_listening_event()` and `update_listening_streak()`. I will reproduce the reported behavior and examine how date differences and calendar-day conditions affect the streak calculation.

3. **Issue #4: Rating a song does not create a notification**  
   I will trace the rating flow from `routes/songs.py` to `notification_service.rate_song()` and compare it with the working notification flow in `add_to_playlist()`. I will reproduce the issue before determining the root cause.

I chose these three issues as my initial plan because they involve different parts of the application and provide opportunities to trace execution across routes, services, models, and database operations. I will reproduce each bug before modifying the code and document the root cause and fix separately.

### Baseline Test Results

Before making any bug fixes, I ran the existing test suite using `pytest tests/`. The baseline result was 10 passing tests and 3 failing tests. Two failures occurred in the playlist tests, where the returned playlist omitted a song and did not match the expected order. One failure occurred in the streak tests, where the listening streak did not increment as expected on Sunday. These baseline failures provide reproducible evidence of Issues #5 and #1 before any code changes.

### AI Assistance Disclosure

I used AI assistance during codebase orientation to help summarize the responsibilities of the main files and trace the rating feature across the route, service, and model layers. I reviewed the source code directly and verified the documented code structure and execution flow against the repository.

## Milestone 2: Reproduce Chosen Bugs Before Fixing

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:**  
I ran the playlist tests using `pytest tests/test_playlists.py -v`. The test suite produced two failures and one passing test. The failing tests were `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order`.

**Observed behavior:**  
The playlist returned 4 songs when 5 songs were expected. The ordered playlist result also omitted the final song.

**Expected behavior:**  
The playlist should return all 5 songs in ascending playlist position order.

**Code changed before reproduction:**  
No.

---

### Issue #1: My listening streak keeps resetting

**How I reproduced it:**  
I ran the streak tests using `pytest tests/test_streaks.py -v`. The test suite produced one failure and four passing tests. The failing test was `test_streak_increments_on_sunday`.

**Observed behavior:**  
The listening streak remained at 1 when it should have incremented to 2 for consecutive-day listening on Sunday.

**Expected behavior:**  
If a user listened on the previous calendar day, the listening streak should increment by 1 even when the current day is Sunday.

**Code changed before reproduction:**  
No.

---

### Issue #4: Rating a song does not create a notification

**How I reproduced it:**  
I used seed data where `Midnight Drive` was shared by `nova`. Before rating, I checked nova's notifications and saw one existing `song_added_to_playlist` notification. Then I had `darius` rate `Midnight Drive` with a score of 5 by calling `rate_song()`. The rating was created successfully.

**Observed behavior:**  
After the rating was created, nova's notification count remained 1. The only notification was still `song_added_to_playlist`. No `song_rated` notification was created.

**Expected behavior:**  
When a user rates a song shared by someone else, the original sharer should receive a `song_rated` notification.

**Code changed before reproduction:**  
No.

## Milestone 3: Root Cause Analysis

### Issue #5: The last song in a playlist never shows up

**1. Issue number and title**

Issue #5: The last song in a playlist never shows up.

**2. How I reproduced it**

I ran `pytest tests/test_playlists.py -v` before changing any source code. The `test_playlist_returns_all_songs` test failed because the function returned 4 songs when 5 were expected. The `test_playlist_returns_songs_in_order` test also failed because the final song was missing from the ordered result.

**3. How I found the root cause**

I traced the execution flow from the `GET /playlists/<playlist_id>/songs` endpoint in `routes/playlists.py` to the `get_songs()` route function and then to `get_playlist_songs()` in `services/playlist_service.py`. The database query correctly retrieved the playlist songs and ordered them by `playlist_entries.c.position`. I then examined the function's return statement and found that it used `songs[:-1]`, which excluded the final element from the query result.

**4. The root cause**

The root cause was the list slice `songs[:-1]` in `get_playlist_songs()`. In Python, `[:-1]` returns every element except the last one. Although the database query retrieved all playlist songs correctly, the return statement removed the final song before converting the results to dictionaries and returning them to the route.

**5. My fix and side-effect check**

I changed the return statement from `songs[:-1]` to `songs`, allowing every song retrieved by the ordered database query to be returned. I then ran `pytest tests/test_playlists.py -v`. All three playlist tests passed, confirming that the function returns all songs, preserves playlist order, and continues to return an empty list for an empty playlist.

### Issue #1: My listening streak keeps resetting

**1. Issue number and title**

Issue #1: My listening streak keeps resetting.

**2. How I reproduced it**

I ran `pytest tests/test_streaks.py -v` before changing any source code. The `test_streak_increments_on_sunday` test failed because the listening streak remained at 1 when it should have incremented to 2 for consecutive-day listening on Sunday.

**3. How I found the root cause**

I traced the execution flow from the `POST /songs/<song_id>/listen` endpoint in `routes/songs.py` to `record_listening_event()` and then to `update_listening_streak()` in `services/streak_service.py`. The function correctly calculated `days_since_last`, but I found an additional condition in the consecutive-day branch that checked `today.weekday() != 6`. Since Python's `weekday()` method returns 6 for Sunday, this condition prevented the streak from incrementing on Sunday.

**4. The root cause**

The root cause was the condition `days_since_last == 1 and today.weekday() != 6`. A user who listened on Saturday and again on Sunday had a `days_since_last` value of 1, but `today.weekday() != 6` evaluated to false because Sunday is represented by 6. The code therefore entered the `else` branch and reset the streak to 1 instead of incrementing it. The day-of-week check was unnecessary because the streak rules depend only on consecutive calendar days.

**5. My fix and side-effect check**

I removed the unnecessary `today.weekday() != 6` condition so that any `days_since_last` value of 1 increments the listening streak. I then ran `pytest tests/test_streaks.py -v`. All five streak tests passed, confirming that the fix handles Sunday correctly while preserving the expected behavior for new users, consecutive-day listening, repeated listening on the same day, and listening after a skipped day.