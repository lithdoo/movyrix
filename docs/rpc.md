# Movyrix RPC Protocol (Draft)

Movyrix uses **JSON-RPC 2.0 over WebSocket** for player control and state synchronization.

This document is an initial protocol draft. Method names and payloads may evolve until the first stable protocol version is declared.

## 1. Transport

Default local transport:

```text
WebSocket
ws://127.0.0.1:<port>/rpc
```

The exact port is implementation-defined during bootstrap.

Each WebSocket text message contains one JSON-RPC 2.0 object. Batch requests are not required for the first implementation.

## 2. Request

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "player.pause"
}
```

A command that accepts parameters:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "player.seek",
  "params": {
    "position": 123.5
  }
}
```

## 3. Successful response

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "ok": true
  }
}
```

Read methods may return structured data directly:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "state": "playing",
    "position": 123.5,
    "duration": 7200.0,
    "rate": 1.0,
    "volume": 80.0,
    "muted": false
  }
}
```

## 4. Error response

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "error": {
    "code": -32001,
    "message": "Media not found",
    "data": {
      "uri": "D:\\Movies\\missing.mkv"
    }
  }
}
```

Suggested application error range:

```text
-32000  Generic player error
-32001  Media not found
-32002  Unsupported or failed media load
-32003  Invalid player state
-32004  Invalid track
-32005  Seek unavailable
-32006  Permission denied
-32007  Backend unavailable
-32008  Invalid media URI
```

Standard JSON-RPC errors should retain their standard meanings.

## 5. Notifications and events

Server-to-client events use JSON-RPC notifications (no `id`).

```json
{
  "jsonrpc": "2.0",
  "method": "event.position",
  "params": {
    "position": 123.7
  }
}
```

Playback-state change:

```json
{
  "jsonrpc": "2.0",
  "method": "event.playbackState",
  "params": {
    "state": "paused"
  }
}
```

Media loaded:

```json
{
  "jsonrpc": "2.0",
  "method": "event.mediaLoaded",
  "params": {
    "duration": 7265.23,
    "title": "Example",
    "video": {
      "width": 3840,
      "height": 2160,
      "codec": "hevc"
    }
  }
}
```

## 6. Player methods

### `player.play`

Resume/start playback.

```json
{
  "jsonrpc": "2.0",
  "id": 10,
  "method": "player.play"
}
```

### `player.pause`

Pause playback.

```json
{
  "jsonrpc": "2.0",
  "id": 11,
  "method": "player.pause"
}
```

### `player.stop`

Stop current playback.

### `player.seek`

Seek to an absolute position in seconds.

```json
{
  "jsonrpc": "2.0",
  "id": 12,
  "method": "player.seek",
  "params": {
    "position": 600.25
  }
}
```

Future versions may add explicit relative seeking instead of overloading this method.

### `player.state`

Return a full state snapshot. This is the primary reconnection/synchronization method.

Example result:

```json
{
  "state": "playing",
  "position": 125.32,
  "duration": 7200.0,
  "rate": 1.0,
  "volume": 80.0,
  "muted": false,
  "media": {
    "uri": "D:\\Movies\\movie.mkv",
    "title": "movie.mkv"
  },
  "video": {
    "width": 3840,
    "height": 2160,
    "codec": "hevc"
  },
  "audioTrack": "audio:2",
  "subtitleTrack": "subtitle:4"
}
```

## 7. Media methods

### `media.open`

Open a media URI.

```json
{
  "jsonrpc": "2.0",
  "id": 20,
  "method": "media.open",
  "params": {
    "uri": "D:\\Movies\\movie.mkv"
  }
}
```

Opening a media item does not require clients to know backend-specific file-loading commands.

### `media.close`

Close the current media.

### `media.info`

Return metadata for the currently loaded media.

## 8. Audio methods

### `audio.volume.set`

```json
{
  "jsonrpc": "2.0",
  "id": 30,
  "method": "audio.volume.set",
  "params": {
    "value": 80.0
  }
}
```

### `audio.mute.set`

```json
{
  "jsonrpc": "2.0",
  "id": 31,
  "method": "audio.mute.set",
  "params": {
    "muted": true
  }
}
```

### `audio.track.set`

```json
{
  "jsonrpc": "2.0",
  "id": 32,
  "method": "audio.track.set",
  "params": {
    "id": "audio:2"
  }
}
```

## 9. Subtitle methods

### `subtitle.track.set`

```json
{
  "jsonrpc": "2.0",
  "id": 40,
  "method": "subtitle.track.set",
  "params": {
    "id": "subtitle:4"
  }
}
```

Use `null` to disable subtitles:

```json
{
  "jsonrpc": "2.0",
  "id": 41,
  "method": "subtitle.track.set",
  "params": {
    "id": null
  }
}
```

### `subtitle.add`

Load an external subtitle resource.

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "method": "subtitle.add",
  "params": {
    "uri": "D:\\Movies\\movie.zh-CN.ass"
  }
}
```

### `subtitle.delay.set`

Set subtitle timing offset in seconds.

## 10. Track discovery

A future `media.tracks` or `player.tracks` method should return normalized track objects.

Example:

```json
{
  "tracks": [
    {
      "id": "video:1",
      "type": "video",
      "codec": "hevc",
      "selected": true
    },
    {
      "id": "audio:2",
      "type": "audio",
      "codec": "aac",
      "language": "ja",
      "title": "Japanese",
      "selected": true
    },
    {
      "id": "subtitle:4",
      "type": "subtitle",
      "codec": "ass",
      "language": "zh-CN",
      "title": "Chinese",
      "selected": true
    }
  ]
}
```

Protocol track IDs are opaque and should not be treated as raw libmpv IDs by clients.

## 11. Playlist methods

Planned methods:

```text
playlist.get
playlist.add
playlist.remove
playlist.move
playlist.next
playlist.previous
```

Exact schemas will be added when playlist behavior is implemented.

## 12. System methods

Planned methods include:

```text
system.info
system.version
system.capabilities
```

`system.capabilities` should eventually expose feature support without leaking backend implementation details.

## 13. Event catalog

Initial planned notifications:

```text
event.connected
event.mediaLoaded
event.mediaClosed
event.playbackState
event.position
event.duration
event.volume
event.rate
event.tracksChanged
event.audioTrackChanged
event.subtitleTrackChanged
event.buffering
event.ended
event.error
```

Not every backend event needs a public RPC event. Only stable domain-level events should cross the protocol boundary.

## 14. Position update policy

The native backend may update playback position far more frequently than clients need.

Recommended server behavior:

- emit `event.position` around 4-10 Hz during normal playback
- emit an immediate position event after a completed seek
- allow clients to interpolate the progress bar between notifications

This keeps UI smooth without generating unnecessary JSON/WebSocket traffic.

## 15. Versioning

The protocol should gain an explicit version before external clients are considered stable.

Possible discovery response:

```json
{
  "name": "Movyrix",
  "protocolVersion": "0.1",
  "serverVersion": "0.1.0"
}
```

Breaking protocol changes should require a protocol-version bump.

## 16. Security

For the local-only phase:

- bind to loopback by default
- validate browser origins
- validate request schema and parameter ranges
- cap message size
- reject arbitrary backend commands/properties
- avoid generic filesystem RPC methods

For LAN/remote access, authentication/pairing must be introduced before non-loopback binding is enabled by default.

## 17. Non-goal

The RPC channel is **not** a media transport.

Do not send decoded video frames, PCM audio, or large media buffers through JSON-RPC/WebSocket. Media rendering remains in the native playback pipeline.
