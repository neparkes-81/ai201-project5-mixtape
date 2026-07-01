
<!-- each bug fix should use a commit in the convention -->
<!-- fix: correct Sunday boundary condition in streak reset logic -->

## Codebase Map
This code base map breaks down files in order of import depth. Thus the first section is highest in the hierarchy and then next is called upon by the one before and so on.

### 1) App
```
ai201-project5-mixtape-starter/
├── app.py                      # Flask app factory and DB setup
```
Connects all routes and starts up app.

### 2) Routes
```
├── routes/
│   ├── songs.py                # Song sharing, search, and rating routes
│   ├── playlists.py            # Playlist creation and song management
│   ├── users.py                # User profiles, streaks, notifications
│   └── feed.py                 # Friends listening now, activity feed
```
Defines routes for requests or posts to the application. Here services are called. Every route delegates immediately to a service function. The routes simply do input parsing and response formatting.

### 3) Services
```
├── services/
│   ├── streak_service.py       # Listening streak logic
│   ├── feed_service.py         # Friends listening now feed logic
│   ├── search_service.py       # Song search logic
│   ├── notification_service.py # Notification creation and retrieval
│   └── playlist_service.py     # Playlist retrieval logic
```
These service files define functions which are used for fundamental app functionality. These functions call on models to manipulate and use data instances stores (playlists, songs, etc.).  


### 4) Models
```
├── models.py                   # SQLAlchemy models for all entities
```
Defines 5 SQLAlchemy models: User, Song, Playlist, PlaylistSong, and Notification.

### Miscellaneous
```
├── tests/
│   ├── test_streaks.py
│   ├── test_search.py
│   └── test_playlists.py
├── seed_data.py                # Populates DB with test data
├── requirements.txt
└── .gitignore
```
Remaining files in the codebase are for testing and set up.

## Root Cause Analysis
### #5 - The last song in a playlist never shows up
<!-- What steps did you take to confirm the bug exists before touching any code? What inputs, sequence of actions, or data condition triggered the behavior? -->
* **How you reproduced it** - using the command `curl http://127.0.0.1:5000/playlists/<playlist_id>/` I could see this error in action. I also used the pytest test suite to cross reference since one of the tests isolated this issue, as well.
<!-- Which files did you look at? What was your navigation path? What moment made you confident you'd found the right place — not just a suspicious area, but the specific cause? -->
* **How you found the root cause** — So I noticed that the route called upon the service function `get_playlist_songs(playlist_id)`. This is supposed to provide a list of dictionaries each representing a song in the playlist. I then navigated to this function definition in the `playlist_service` file. I was confident the issue was here when I noticed an problem with the way the list was created.
<!-- In plain English, explain exactly what was wrong. Not "there was a bug in the streak logic" — explain the specific condition, comparison, or missing step that caused the problem. -->
* **The root cause** — The cause of the bug was found in the list comprehension step of the `get_playlist_songs(playlist_id)` function. The list being created was pulling songs from `songs[:-1]`. This problematic because this would mean the function is returning including songs in the originally retrieved list up to the -1th, or last, list item, non-inclusive.

<!-- What did you change and why does that change fix the root cause? What related functionality did you check afterward to confirm you didn't break anything? -->
* **Your fix and side-effect check** — I took away the exclusion of the last song and simply had it take from the original list `songs`. I checked this fix with the test suite and everything worked fine.


## AI Usage

