# MusicKit JS Troubleshooting

## Common Errors

### "MusicKit is not defined"

**Cause**: The SDK script hasn't loaded yet.

**Solution**: Wait for the SDK to load before initializing:

```javascript
connect() {
  if (!window.MusicKit) {
    setTimeout(() => this.connect(), 1000)
    return
  }
  this.initializeMusicKit()
}
```

Or use the SDK's ready callback:

```javascript
document.addEventListener("musickitloaded", () => {
  this.initializeMusicKit()
})
```

### "Refused to load the script https://js-cdn.music.apple.com/..."

**Cause**: Content Security Policy blocks the CDN script.

**Solution**: Allow the CDN and API domains in your CSP config:

```ruby
policy.script_src :self, :https, "https://js-cdn.music.apple.com"
policy.connect_src :self, :https, "https://api.music.apple.com"
```

### "MusicKit.configure is not a function"

**Cause**: The wrong SDK URL is loaded or the script was blocked.

**Solution**: Verify you are loading v3 from `https://js-cdn.music.apple.com/musickit/v3/musickit.js` and check the console for CSP/network errors.

### "Developer token is invalid"

**Causes**:
1. Token expired (max 180 days)
2. Wrong algorithm (must be ES256)
3. Key ID doesn't match the private key
4. Team ID is incorrect

**Debug steps**:

```ruby
# In Rails console
token = Apple::MusicKitToken.get
payload = JWT.decode(token, nil, false).first

# Check expiration
Time.at(payload["exp"])  # Should be in the future

# Check issuer matches your Team ID
payload["iss"]  # Should be your 10-char Team ID
```

### "Unauthorized" - User Not Authorized

**Cause**: User hasn't granted Apple Music access.

**Solution**: Call `music.authorize()` and handle the UI state:

```javascript
if (!music.isAuthorized) {
  this.showAuthPrompt()
}
```

### "Playback only plays previews"

**Cause**: The user is not authorized or does not have an active Apple Music subscription.

**Solution**: Prompt authorization with `music.authorize()` and keep preview fallback for users without subscriptions.

### "This content is not available"

**Causes**:
1. Content restricted in user's region
2. Album/track removed from Apple Music
3. Invalid Apple Music ID

**Solution**: Handle gracefully with user feedback:

```javascript
async loadAlbum(appleId) {
  try {
    await music.setQueue({ album: appleId })
    await music.play()
  } catch (error) {
    this.showError("This album is not available in your region")
  }
}
```

### Token Generation Errors

#### "PEM_read_bio_PrivateKey: bad base64 decode"

**Cause**: Private key file is corrupted or has wrong format.

**Solution**: The `.p8` file should look like:

```
-----BEGIN PRIVATE KEY-----
MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQg...
-----END PRIVATE KEY-----
```

Verify with:

```bash
openssl ec -in config/keys/AuthKey_XXXXX.p8 -text -noout
```

#### "wrong tag" or "unsupported algorithm"

**Cause**: Key is not an EC key or wrong curve.

**Solution**: Apple requires ES256 (ECDSA with P-256 curve). Check your key:

```ruby
key = OpenSSL::PKey::EC.new(File.read("path/to/key.p8"))
key.group.curve_name  # Should be "prime256v1"
```

## Debugging Tips

### Log MusicKit Events

```javascript
setupDebugListeners() {
  const events = [
    "playbackStateDidChange",
    "nowPlayingItemDidChange",
    "authorizationStatusDidChange",
    "queueItemsDidChange",
    "queuePositionDidChange"
  ]

  events.forEach(event => {
    music.addEventListener(event, (e) => {
      console.log(`[MusicKit] ${event}:`, e)
    })
  })
}
```

### Check Playback State

```javascript
debugPlaybackState() {
  const music = window.MusicKitInstance
  console.log({
    isPlaying: music.isPlaying,
    isAuthorized: music.isAuthorized,
    playbackState: music.playbackState,
    nowPlaying: music.nowPlayingItem?.name,
    queueLength: music.queue?.items?.length,
    currentTime: music.currentPlaybackTime,
    duration: music.currentPlaybackDuration
  })
}
```

### Verify Token in Browser

```javascript
// In browser console
fetch("/api/musickit/token")
  .then(r => r.json())
  .then(({ token }) => {
    // Decode without verification to inspect
    const [header, payload] = token.split(".").slice(0, 2)
    console.log("Header:", JSON.parse(atob(header)))
    console.log("Payload:", JSON.parse(atob(payload)))
  })
```

## Performance Considerations

### Avoid Re-initializing MusicKit

MusicKit.configure() should only be called once. Subsequent calls may cause issues:

```javascript
// Good - check for existing instance
if (!window.MusicKitInstance) {
  window.MusicKitInstance = await MusicKit.configure({...})
}

// Bad - configuring multiple times
await MusicKit.configure({...})  // Don't do this repeatedly
```

### Token Caching

Don't fetch a new token on every page load. Cache it:

```ruby
# Token is cached for 180 days
Rails.cache.write(CACHE_KEY, token, expires_in: 180.days)
```

### Event Listener Cleanup

When using Stimulus, avoid duplicate event listeners:

```javascript
connect() {
  // Store handler reference for cleanup
  this.playHandler ||= (e) => this.handlePlay(e)
  window.addEventListener("musickit:play", this.playHandler)
}

disconnect() {
  window.removeEventListener("musickit:play", this.playHandler)
}
```

## MusicKit SDK Versions

Always use v3 of the SDK:

```html
<script src="https://js-cdn.music.apple.com/musickit/v3/musickit.js" async></script>
```

Version differences:
- **v1**: Legacy, deprecated
- **v2**: Stable but missing features
- **v3**: Current, full feature set including queue management

## Testing Without Apple Music Subscription

- You can test authorization flow without a subscription
- Playback will be limited to 30-second previews
- Full playback requires the user to have an active Apple Music subscription
