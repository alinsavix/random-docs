# Nightbot Song Requests API — Findings

Source file: `index-DW5xu8tp.js` (minified/bundled frontend JS)

---

## Base URL & Versioning

```
https://api.nightbot.tv
```

All REST paths are prefixed with the API version: `/1/`. So a path like
`/song_requests/queue` becomes:

```
https://api.nightbot.tv/1/song_requests/queue
```

---

## Authentication

Every request carries two headers:

| Header | Value | Notes |
|--------|-------|-------|
| `Authorization` | `Session <accessToken>` | OAuth session token obtained at login |
| `Nightbot-Channel` | `<channelId>` | The channel's internal `_id`; required for all channel-scoped operations |
| `Content-Type` | `application/json` | Added automatically when a request body is present |

---

## REST Endpoints

### Settings

| Method | Path | Description |
|--------|------|-------------|
| `GET`  | `/1/song_requests` | Get current settings, available providers, and playlists |
| `PUT`  | `/1/song_requests` | Update settings |

**GET response body:**
```json
{
  "settings": { ... },
  "providers": { ... },
  "playlists": [ ... ]
}
```

**Settings object** (the body sent on `PUT`, and the `settings` field returned by `GET`):
```json
{
  "enabled": false,
  "providers": ["youtube", "soundcloud"],
  "playlist": "channel",
  "userLevel": "everyone",
  "searchProvider": "youtube",
  "youtube": {
    "limitToMusic": false,
    "limitToLikedVideos": false
  },
  "limits": {
    "queue": 20,
    "user": 5,
    "playlistOnly": false,
    "exemptUserLevel": "moderator"
  },
  "volume": 50
}
```

| Field | Type | Notes |
|-------|------|-------|
| `enabled` | boolean | Whether song requests are on |
| `providers` | string[] | Enabled providers; valid values: `"youtube"`, `"soundcloud"` |
| `playlist` | string | Active playlist source: `"channel"` or `"monstercat"` |
| `userLevel` | string | Minimum chat role to request; values: `"everyone"`, `"regular"`, `"subscriber"`, `"moderator"`, etc. |
| `searchProvider` | string | Default provider used when a user types a title (not a URL); same values as `providers` |
| `youtube.limitToMusic` | boolean | Restrict YouTube requests to music category |
| `youtube.limitToLikedVideos` | boolean | Restrict YouTube requests to the channel owner's liked videos |
| `limits.queue` | integer | Max total songs in queue at once (default 20) |
| `limits.user` | integer | Max songs any one user can have in queue (default 5) |
| `limits.playlistOnly` | boolean | When true, only songs already in the playlist can be requested |
| `limits.exemptUserLevel` | string | Users at this level or above are exempt from queue/user limits |
| `volume` | integer | Playback volume 0–100 (default 50) |

---

### Queue

| Method | Path | Description |
|--------|------|-------------|
| `GET`    | `/1/song_requests/queue` | Get the full queue and currently playing song |
| `POST`   | `/1/song_requests/queue` | Add a song to the queue |
| `DELETE` | `/1/song_requests/queue` | Clear the entire queue |
| `DELETE` | `/1/song_requests/queue/:id` | Remove one item by its `_id` |
| `POST`   | `/1/song_requests/queue/:id/play` | Skip to / immediately play a specific queue item |
| `POST`   | `/1/song_requests/queue/:id/promote` | Promote an item to the top of the queue (next to play) |
| `PATCH`  | `/1/song_requests/queue/order` | Reorder the entire queue |

#### GET `/1/song_requests/queue`

Response:
```json
{
  "queue": [ { ...songItem }, ... ],
  "_currentSong": { ...songItem }
}
```

`_currentSong` is the song currently playing (may be `null` if nothing is playing).

#### POST `/1/song_requests/queue` — Add a song by search or URL

Request body:
```json
{ "q": "search query or URL" }
```

If hCaptcha is required (server signals this), re-submit with:
- Query string: `?has_captcha=true`
- Body: `{ "q": "...", "captcha": "<hCaptcha token>" }`

Response:
```json
{ "item": { ...songItem } }
```

#### POST `/1/song_requests/queue` — Queue a song from the playlist

Request body:
```json
{ "fromPlaylist": true, "q": "<playlist_item_id>" }
```

Response:
```json
{ "item": { ...songItem } }
```

#### POST `/1/song_requests/queue/:id/play`

No request body. Immediately begins playing the specified queue item (skipping whatever is current).

Response:
```json
{ "item": { ...songItem } }
```

#### POST `/1/song_requests/queue/:id/promote`

No request body. Moves the specified item to position 1 in the queue (next to play after the current song).

#### PATCH `/1/song_requests/queue/order`

Request body — send the full desired ordering as an array of `_id` values:
```json
{ "order": ["id1", "id2", "id3"] }
```

---

### Playlist

| Method | Path | Description |
|--------|------|-------------|
| `GET`    | `/1/song_requests/playlist` | Get playlist (paginated) |
| `POST`   | `/1/song_requests/playlist` | Add a song to the playlist |
| `DELETE` | `/1/song_requests/playlist` | Clear the entire playlist |
| `DELETE` | `/1/song_requests/playlist/:id` | Remove one item by its `_id` |
| `POST`   | `/1/song_requests/playlist/import` | Bulk-import a YouTube playlist by URL |

#### GET `/1/song_requests/playlist`

Supported query parameters (all optional):

| Param | Description |
|-------|-------------|
| `offset` | Zero-based offset for pagination |
| `limit` | Number of items to return |

Response:
```json
{
  "playlist": [ { ...songItem }, ... ],
  "_total": 142,
  "_offset": 0,
  "_limit": 20
}
```

#### POST `/1/song_requests/playlist` — Add a song

Request body:
```json
{ "q": "search query or URL" }
```

Response:
```json
{ "item": { ...songItem } }
```

#### POST `/1/song_requests/playlist/import` — Import a playlist

Request body:
```json
{ "url": "https://www.youtube.com/playlist?list=..." }
```

No meaningful response body (fire-and-forget; import runs server-side).

---

## Error Handling

All errors return a JSON body:
```json
{ "message": "Human-readable description" }
```

| Status | Behaviour |
|--------|-----------|
| `429 Too Many Requests` | Client auto-retries after the number of seconds in the `retry-after` response header |
| `401 Unauthorized` | Session is invalidated client-side and the user is logged out |

---

## Retrieving Another Streamer's Queue (Public, No Auth)

The queue and playlist for any channel are publicly readable without authentication.
The two-step process mirrors what the nightbot.tv website itself does when you visit
`nightbot.tv/:provider/:username/song_requests`.

### Step 1 — Resolve the channel `_id` from provider + username

```
GET https://api.nightbot.tv/1/channels/:provider/:username
```

| Segment | Values |
|---------|--------|
| `:provider` | `twitch`, `youtube`, `trovo`, `soop_global`, `soop` |
| `:username` | The streamer's username on that platform |

No authentication headers required.

Response:
```json
{ "channel": { "_id": "abc123...", ... } }
```

### Step 2 — Fetch the queue using the `_id`

```
GET https://api.nightbot.tv/1/song_requests/queue
Nightbot-Channel: abc123...
```

Response:
```json
{
  "queue": [ { ...songItem }, ... ],
  "_currentSong": { ...songItem }
}
```

`_currentSong` is `null` if nothing is currently playing.

### Example (curl)

```sh
# Step 1 — resolve channel _id for a Twitch streamer
curl https://api.nightbot.tv/1/channels/twitch/examplestreamer

# Step 2 — fetch their song request queue
curl https://api.nightbot.tv/1/song_requests/queue \
  -H "Nightbot-Channel: <_id from step 1>"
```

The `Authorization` header is **not** needed for these two read operations.
It is only required for write operations (adding/removing/reordering songs, updating settings)
or for reading data scoped to your own account.

---

## WebSocket — Real-time Push Updates

The queue page keeps itself live using a plain WebSocket (not Socket.IO). The server pushes events whenever queue state changes, so the UI never needs to poll.

### Step 1 — Obtain a short-lived token

```
GET https://api.nightbot.tv/1/me/ws_token
Authorization: Session <accessToken>
Nightbot-Channel: <channelId>
```

Response:
```json
{ "token": "<short-lived token string>" }
```

### Step 2 — Open the WebSocket

```
wss://ws.nightbot.tv/ws?token=<token>&channel=<channelId>
```

### Step 3 — Receive messages

Every message is a JSON-encoded object:
```json
{ "event": "<eventName>", "data": { ... } }
```

#### Song-request events

| Event | Meaning |
|-------|---------|
| `songRequestQueueAdd` | A song was added to the queue |
| `songRequestQueueRemove` | A queue item was removed |
| `songRequestQueueClear` | The entire queue was cleared |
| `songRequestQueuePromote` | A queue item was promoted to the top |
| `songRequestSkip` | The current song was skipped |
| `songRequestPlay` | Playback started or changed to a new song |
| `songRequestPause` | Playback was paused |
| `songRequestVolume` | Volume was changed |

### Reconnection

The client uses exponential backoff to reconnect automatically on `close` or `error`. A fresh token is fetched from `/1/me/ws_token` on each reconnect attempt.
