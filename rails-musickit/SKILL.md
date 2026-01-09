---
name: rails-musickit
description: Apple MusicKit JS SDK integration for Rails 8 (importmap + Stimulus). Use when adding the MusicKit script tag, generating ES256 developer JWT tokens, building /api/musickit/token endpoints, wiring player/trigger controllers, handling authorization or user tokens, or troubleshooting Apple Music playback.
---

# Apple MusicKit JS + Rails 8 Integration

Integrate MusicKit JS into Rails 8 with importmaps and Stimulus controllers.

## Constraints

- Load MusicKit from Apple's CDN via a `<script>` tag; do not bundle via npm/importmap.
- Developer tokens must be ES256 JWTs and valid for <= 180 days.
- Configure MusicKit once and reuse the singleton across Turbo visits.
- Users must authorize to play full tracks; previews can play without authorization.
- If CSP is enabled, allow `https://js-cdn.music.apple.com` in `script-src` and `https://api.music.apple.com` in `connect-src`.

## Quick Reference

| Component | Location |
|-----------|----------|
| SDK script tag | `app/views/layouts/application.html.erb` |
| Player UI container | `app/views/nav/_apple_music_player.html.erb` |
| Player placement | `app/views/nav/_sidebar.html.erb` |
| Player controller | `app/javascript/controllers/apple_musickit_player_controller.js` |
| Trigger controller | `app/javascript/controllers/apple_musickit_trigger_controller.js` |
| Stimulus registry | `app/javascript/controllers/index.js` |
| Token generator | `app/lib/apple/music_kit_token.rb` |
| Token endpoint | `app/controllers/api/music_kit_controller.rb` |
| Routes | `config/routes.rb` |
| Play button partial | `app/views/albums/_apple_play_button.html.erb` |
| Track list partial | `app/views/albums/_apple_tracks.html.erb` |

## Step 1: Load the SDK

Add the MusicKit SDK to your layout (do not pin in importmap):

```erb
<%# app/views/layouts/application.html.erb %>
<script src="https://js-cdn.music.apple.com/musickit/v3/musickit.js" async></script>
```

Wait for the SDK before calling `MusicKit.configure` (use the `musickitloaded` event or a `window.MusicKit` check).

## Step 2: Generate Developer Tokens

Store the `.p8` key under `config/keys/AuthKey_<KEY_ID>.p8`, keep it out of Git, and supply Team ID, Key ID, and Music Identifier via env/credentials. Cache tokens and refresh early.

```ruby
# app/lib/apple/music_kit_token.rb
module Apple
  class MusicKitToken
    MAX_TOKEN_AGE = 180.days.freeze
    CACHE_KEY = "apple_musickit_token".freeze

    class ConfigError < StandardError; end

    def self.get
      valid? ? Rails.cache.read(CACHE_KEY) : new.generate
    end

    def self.valid?
      cached_token = Rails.cache.read(CACHE_KEY)
      return false if cached_token.blank?

      payload = JWT.decode(cached_token, nil, false).first
      Time.zone.at(payload["exp"]) > 1.week.from_now
    rescue JWT::DecodeError
      false
    end

    def initialize
      validate_config!
    end

    def generate
      token = encode_token
      Rails.cache.write(CACHE_KEY, token, expires_in: MAX_TOKEN_AGE)
      token
    end

    private

    def validate_config!
      raise ConfigError, "Team ID is required" if team_id.blank?
      raise ConfigError, "Key ID is required" if key_id.blank?
      raise ConfigError, "MusicKit ID is required" if identifier.blank?
      raise ConfigError, "Private key not found at #{key_path}" if private_key.blank?
    end

    def encode_token
      now = Time.now.to_i
      payload = {
        iss: team_id,
        iat: now,
        exp: now + MAX_TOKEN_AGE.to_i,
        sub: identifier
      }

      JWT.encode(payload, ec_key, "ES256", kid: key_id)
    end

    def ec_key
      @ec_key ||= OpenSSL::PKey::EC.new(private_key)
    end

    def private_key
      @private_key ||= File.read(key_path)
    rescue Errno::ENOENT
      nil
    end

    def key_path
      Rails.root.join("config/keys/AuthKey_#{key_id}.p8")
    end

    def team_id = ENV.fetch("APPLE_TEAM_ID", nil)
    def key_id = ENV.fetch("APPLE_MUSICKIT_KEY_ID", nil)
    def identifier = ENV.fetch("APPLE_MUSICKIT_IDENTIFIER", nil)
  end
end
```

## Step 3: Expose a Token Endpoint

```ruby
# app/controllers/api/music_kit_controller.rb
class API::MusicKitController < ActionController::API
  def token
    render json: { token: Apple::MusicKitToken.get }
  rescue Apple::MusicKitToken::ConfigError => e
    render json: { error: e.message }, status: :unprocessable_content
  end
end
```

```ruby
# config/routes.rb
namespace :api do
  get "musickit/token", to: "music_kit#token"
end
```

## Step 4: Wire Player + Trigger Controllers

Place a single player controller in a persistent UI region (sidebar/nav) and dispatch play requests via a trigger controller on buttons or track rows.

Example trigger markup:

```erb
<button
  type="button"
  data-controller="apple-musickit-trigger"
  data-apple-musickit-trigger-apple-id-value="123"
  data-apple-musickit-trigger-resource-type-value="album"
  data-apple-musickit-trigger-track-id-value="456"
  data-apple-musickit-trigger-preview-url-value="https://example.com/preview.m4a"
  data-action="click->apple-musickit-trigger#play">
  Play
</button>
```

Dispatch a `musickit:play` event with a detail payload that includes `appleId`, `resourceType`, and optional preview metadata. Keep listeners stable across Turbo visits to avoid duplicate handlers.

## Step 5: Handle Authorization + User Tokens

Call `music.authorize()` on user action, update UI from `music.isAuthorized`, and use `music.musicUserToken` (or the resolved token) if you call Apple Music APIs on behalf of the user.

For detailed implementation patterns, see [IMPLEMENTATION.md](IMPLEMENTATION.md).

For common issues and debugging, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## Apple Developer Setup

1. Go to [Apple Developer Portal](https://developer.apple.com/account/resources/authkeys/list)
2. Create a MusicKit key (type: Media Services)
3. Download the `.p8` private key file
4. Note your Team ID, Key ID, and create a Music Identifier

Required credentials:
- Team ID: Your 10-character Apple Developer Team ID
- Key ID: The key ID from the MusicKit key you created
- Music Identifier: A reverse-domain identifier (e.g., `media.com.yourapp`)
- Private Key: The `.p8` file contents
