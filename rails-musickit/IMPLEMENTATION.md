# MusicKit JS Implementation Patterns

## SDK Load Handling

Wait for the SDK before calling `MusicKit.configure`. Prefer the `musickitloaded` event and fall back to a `window.MusicKit` check.

```javascript
connect() {
  if (!window.MusicKit) {
    document.addEventListener("musickitloaded", () => this.connect(), { once: true })
    return
  }

  this.initializeMusicKit()
}
```

## MusicKit Instance Management

Always use a singleton pattern for the MusicKit instance:

```javascript
async initializeMusicKit() {
  if (window.MusicKitInstance) return window.MusicKitInstance

  // Fetch token from your backend
  const response = await fetch("/api/musickit/token")
  const { token } = await response.json()

  // Configure MusicKit (only once)
  const music = await window.MusicKit.configure({
    developerToken: token,
    app: {
      name: "Your App Name",
      build: "1.0"
    }
  })

  // Store globally for access across controllers
  window.MusicKitInstance = music
  return music
}
```

## Trigger Event Contract

Dispatch a `musickit:play` CustomEvent from trigger elements. Pass enough detail to support preview playback and UI updates.

```javascript
window.dispatchEvent(new CustomEvent("musickit:play", {
  detail: {
    appleId,
    resourceType, // "album" or "track"
    trackId,
    previewUrl,
    trackName,
    artistName,
    albumName,
    durationMs,
    artworkUrl
  },
  bubbles: true
}))
```

## Playback Queue Operations

### Load an Album

```javascript
async loadAlbum(appleId, startTrackId = null) {
  const music = window.MusicKitInstance

  await music.setQueue({ album: appleId })

  // Optionally start at a specific track
  if (startTrackId) {
    const track = music.queue.items.find(t => t.id === startTrackId)
    if (track) await music.changeToMediaItem(track)
  }

  await music.play()
}
```

### Load a Single Track

```javascript
async loadTrack(songId) {
  const music = window.MusicKitInstance
  await music.setQueue({ song: songId })
  await music.play()
}
```

### Load a Playlist

```javascript
async loadPlaylist(playlistId) {
  const music = window.MusicKitInstance
  await music.setQueue({ playlist: playlistId })
  await music.play()
}
```

## Event Handling

```javascript
setupEventListeners() {
  const music = window.MusicKitInstance
  if (!music || this.boundMusicInstance === music) return
  this.boundMusicInstance = music

  // Playback state (playing, paused, stopped)
  music.addEventListener("playbackStateDidChange", () => {
    console.log("Playing:", music.isPlaying)
  })

  // Track changed
  music.addEventListener("nowPlayingItemDidChange", () => {
    const item = music.nowPlayingItem
    console.log("Now playing:", item?.name)
  })

  // Authorization status changed
  music.addEventListener("authorizationStatusDidChange", () => {
    console.log("Authorized:", music.isAuthorized)
  })

  // Playback time updated (use for progress bars)
  music.addEventListener("playbackTimeDidChange", () => {
    const current = music.currentPlaybackTime
    const duration = music.currentPlaybackDuration
    const percent = (current / duration) * 100
  })
}
```

## User Authorization

Users must authorize to play full tracks. Without authorization, only 30-second previews are available.

```javascript
async authorize() {
  const music = window.MusicKitInstance
  try {
    const userToken = await music.authorize()
    return userToken
  } catch (error) {
    // User denied or closed the dialog
  }
}

checkAuthorization() {
  const music = window.MusicKitInstance
  return music.isAuthorized
}

getUserToken() {
  const music = window.MusicKitInstance
  return music.musicUserToken
}
```

Use the resolved token or `music.musicUserToken` for Apple Music API requests on behalf of the user.

## Preview Playback (Optional)

Use preview URLs when the user is not authorized or lacks a subscription.

```javascript
playPreview(url) {
  this.previewAudio ||= new Audio()
  this.previewAudio.src = url
  return this.previewAudio.play()
}

shouldUsePreview() {
  const music = window.MusicKitInstance
  return !music?.isAuthorized
}
```

## Turbo/Hotwire Compatibility

MusicKit state persists across Turbo navigations. Handle reconnection properly:

```javascript
connect() {
  // Check if MusicKit SDK is loaded
  if (!window.MusicKit) {
    setTimeout(() => this.connect(), 1000)
    return
  }

  // Reuse existing instance if available
  if (window.MusicKitInstance) {
    this.setupEventListeners()
    this.updateUI()
    return
  }

  this.initializeMusicKit()
}

disconnect() {
  // Don't destroy MusicKit - just clean up local listeners
  if (this.progressInterval) {
    clearInterval(this.progressInterval)
  }
}
```

## Persisting Playback Across Page Navigation

Store playback state in sessionStorage for Turbo page transitions:

```javascript
persistPlaybackState() {
  const music = window.MusicKitInstance
  if (!music?.isPlaying) return

  const item = music.nowPlayingItem
  if (!item?.id) return

  sessionStorage.setItem("musickit:resume", JSON.stringify({
    id: item.id,
    time: music.currentPlaybackTime
  }))
}

async resumePlayback() {
  const data = sessionStorage.getItem("musickit:resume")
  if (!data) return

  const { id, time } = JSON.parse(data)
  const music = window.MusicKitInstance

  await music.setQueue({ song: id })
  if (time > 0) await music.seekToTime(time)
  await music.play()

  sessionStorage.removeItem("musickit:resume")
}
```

## Artwork URLs

MusicKit artwork URLs contain placeholders for dimensions:

```javascript
getArtworkUrl(item, size = 300) {
  const url = item.artwork?.url
  if (!url) return null

  return url.replace("{w}", size).replace("{h}", size)
}
```

## Track Duration

Duration is returned in milliseconds from the queue, seconds from playback:

```javascript
// From queue item (milliseconds)
const durationMs = track.playbackDuration || track.durationInMillis
const seconds = durationMs / 1000

// From current playback (seconds)
const seconds = music.currentPlaybackDuration

// Format as mm:ss
function formatTime(seconds) {
  const mins = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${mins}:${secs.toString().padStart(2, "0")}`
}
```

## Seeking

```javascript
seekToPosition(percent) {
  const music = window.MusicKitInstance
  const duration = music.currentPlaybackDuration
  const time = percent * duration
  music.seekToTime(time)
}

// From click event on progress bar
handleProgressClick(event) {
  const bar = event.currentTarget
  const percent = event.offsetX / bar.offsetWidth
  this.seekToPosition(percent)
}
```

## Skip Controls

```javascript
skipToNext() {
  window.MusicKitInstance.skipToNextItem()
}

skipToPrevious() {
  window.MusicKitInstance.skipToPreviousItem()
}
```
