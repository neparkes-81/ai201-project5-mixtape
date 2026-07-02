
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

### #1	- My listening streak keeps resetting
<!-- What steps did you take to confirm the bug exists before touching any code? What inputs, sequence of actions, or data condition triggered the behavior? -->
* **How you reproduced it** - using the command `curl http://127.0.0.1:5000/<user_id>/streak` I could see this error in action. I also used the pytest test suite to cross reference since one of the tests isolated this issue. The pytest also revealed that the issue seems to arise when going between weekend days.
<!-- Which files did you look at? What was your navigation path? What moment made you confident you'd found the right place — not just a suspicious area, but the specific cause? -->
* **How you found the root cause** — So I started in `users.py` to observe the routes which connect to the streak_service functions. I noticed that the streak was obtain by the function `get_streak(user_id)` so I jumped to that function definition in `streak_service.py`. The issue wasn't there, but I was able to find it in `update_listening_streak()`, where I was certain the relevant logic was held.
<!-- In plain English, explain exactly what was wrong. Not "there was a bug in the streak logic" — explain the specific condition, comparison, or missing step that caused the problem. -->
* **The root cause** — The cause of the bug was found in the logic for incrementing streak. It was originally looking at if it has been only one day since the user last listened to something, but also intentionally excluding the scenario when the current day is "day 6".

<!-- What did you change and why does that change fix the root cause? What related functionality did you check afterward to confirm you didn't break anything? -->
* **Your fix and side-effect check** — I removed exclusion of "day 6" from the if statement and simplified the conditional to increment when days since last listened is 1. This is the logical condition to avoid unnecessary skipping.

### #4	- I got notified when a friend added my song to a playlist but not when they rated it
<!-- What steps did you take to confirm the bug exists before touching any code? What inputs, sequence of actions, or data condition triggered the behavior? -->
* **How you reproduced it** - using the command `curl -X POST http://127.0.0.1:5000/songs/<song_id>/rate` to rate a song shared by another user I could see this error in action. When I then checked that user's notifications with `curl http://127.0.0.1:5000/users/<user_id>/notifications` nothing showed up, even though doing the same thing by adding the song to a playlist did create a notification. That difference confirmed the bug existed.
<!-- Which files did you look at? What was your navigation path? What moment made you confident you'd found the right place — not just a suspicious area, but the specific cause? -->
* **How you found the root cause** — So I started in `users.py` to observe the routes which connect to notifications. I noticed that the notification was obtained by the function `get_notification()` so I jumped to that file `notification_service.py`. I noticed that there was specific functions for both add to a playlist and rating one and compared them. It was clear there was an issue with notification when I saw they handled this differently.
<!-- In plain English, explain exactly what was wrong. Not "there was a bug in the streak logic" — explain the specific condition, comparison, or missing step that caused the problem. -->
* **The root cause** — The cause of the bug was the fact that there was no notification logic in the function for rating a song, unlike that what can be found in the function for adding a song to a playlist.

<!-- What did you change and why does that change fix the root cause? What related functionality did you check afterward to confirm you didn't break anything? -->
* **Your fix and side-effect check** — I added logic to notify a user if there song was rated by someone else to the `rate_song()` function. I checked the notification output with print statements when a song is rated.


## AI Usage
<!-- how you used AI tools during codebase navigation and debugging, what they helped you understand, and where you verified or overrode their output -->
I mostly used AI to help me get around the codebase quicker and to double check my thinking. Since every route delegates to a service function, I leaned on it to trace which service a route was actually calling so I could jump straight to the right file instead of reading through everything. It also helped me understand the streak logic, specifically why excluding "day 6" was causing the reset, and it confirmed the way the playlist list was being sliced with `songs[:-1]`. In each case I still went and looked at the code myself to make sure the explanation matched what was really there before I changed anything.

After making my fixes I used AI to verify they actually worked and didn't break anything else. For the streak and playlist bugs I cross referenced with the pytest suite, and for the rating notification I had it confirm the notification was created for the right user and that a user wouldn't get notified for rating their own song. One place I made my own call was on the rating notification, where AI pointed out that re-rating a song would send another notification each time. I decided to leave it as is since it matched how the add to playlist function already behaved, so the two stay consistent.
