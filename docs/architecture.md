# Movyrix Architecture

## 1. Overview

Movyrix is designed around a strict separation between the user interface, the player control protocol, and the native playback engine.

```text
+---------------------------+
| Client UI                 |
| Flutter / Web / Remote    |
+-------------+-------------+
              |
              | WebSocket + JSON-RPC 2.0
              v
+---------------------------+
| Player Service            |
|                           |
| Connection Manager        |
| JSON-RPC Dispatcher       |
| Player Service API        |
| State Store / Event Bus   |
+-------------+-------------+
              |
              | backend-neutral player API
              v
+---------------------------+
| Playback Adapter          |
| initially: libmpv         |
+-------------+-------------+
              |
              v
+---------------------------+
| Native Media Pipeline     |
| FFmpeg / libass / HWDEC   |
| GPU / Audio Device        |
+---------------------------+
```

The important boundary is that RPC transports **commands and state**, not decoded media frames.

## 2. Control plane vs. media plane

### Control plane

The control plane uses WebSocket + JSON-RPC 2.0 and carries small messages such as:

- open/close media
- play/pause/stop
- seek
- volume/mute
- playback rate
- audio track selection
- subtitle track selection
- playlist operations
- playback state
- metadata
- errors and lifecycle events

The transport is intentionally simple enough to support local clients first and authenticated LAN clients later.

### Media plane

The media plane stays native. It owns:

- demuxing
- codec decoding
- subtitle rendering/composition
- audio decoding/output
- hardware decoder selection
- GPU rendering
- synchronization between audio/video/subtitles

Decoded frames should not travel through WebSocket, Dart objects, or JavaScript IPC.

## 3. Platform rendering model

### Windows

Preferred path:

```text
File
 -> libmpv / FFmpeg
 -> hardware decoder (for example D3D11VA when available)
 -> GPU rendering
 -> native/player video surface
 -> display
```

### Android

Preferred path:

```text
File
 -> libmpv / FFmpeg
 -> MediaCodec/native decoding path when available
 -> Android/native surface
 -> display
```

The exact rendering adapter may differ by platform. The Player Service and RPC protocol must not depend on these platform-specific details.

## 4. Player abstraction

The RPC layer should call a backend-neutral interface rather than libmpv directly.

Conceptually:

```text
PlayerService
     |
     v
Player interface
     |
     +---- MpvPlayer
     +---- Future backend
     +---- FakePlayer (tests)
```

A conceptual interface:

```text
open(uri)
close()
play()
pause()
stop()
seek(position)
setVolume(value)
setMuted(value)
setRate(value)
selectAudioTrack(id)
selectSubtitleTrack(id)
addSubtitle(uri)
getState()
getTracks()
```

The public API should avoid leaking backend property names such as `aid`, `sid`, or `time-pos`.

## 5. State ownership

The Player Core is the single source of truth.

Clients should not independently assume that a requested operation succeeded.

Preferred flow:

```text
User presses Pause
       |
       v
player.pause request
       |
       v
Player Core pauses playback
       |
       v
event.playbackState
       |
       v
Client updates UI
```

On connection/reconnection, a client should request `player.state` to obtain a complete snapshot before processing incremental events.

## 6. Event flow

Player backends typically generate many native events. These should be normalized before reaching clients.

Examples:

```text
backend property change
        |
        v
Player Adapter
        |
        v
normalized domain event
        |
        v
State Store / Event Bus
        |
        v
JSON-RPC notification
```

The service may coalesce high-frequency events. For example, playback position normally only needs to be sent around 4-10 times per second.

## 7. Seeking

Dragging a seek bar can generate hundreds of pointer events. The protocol should not blindly translate every pointer movement into a native seek.

Recommended behaviors:

- Default: update the client slider locally and issue one `player.seek` when dragging ends.
- Preview mode: throttle seeks to a bounded interval (for example 50-100 ms) and discard stale seek intents.

The latest user intent should win.

## 8. Track model

Tracks should be represented using a backend-neutral structure, for example:

```json
{
  "id": "audio:2",
  "type": "audio",
  "codec": "aac",
  "language": "ja",
  "title": "Japanese",
  "selected": true,
  "default": false,
  "external": false
}
```

Track IDs are opaque protocol identifiers. Clients must not rely on them being raw mpv track numbers.

## 9. Security model

The initial server should bind to loopback by default:

```text
127.0.0.1
::1
```

Even on localhost, the service should be designed with browser-origin attacks in mind.

Recommended baseline:

- validate `Origin` for browser clients
- avoid exposing arbitrary filesystem operations
- validate all RPC parameters
- limit message size
- limit concurrent connections
- reject unknown methods
- never execute arbitrary mpv commands supplied by a client

Before LAN access is enabled, add pairing/authentication and explicit bind-address configuration.

## 10. Failure handling

Backend errors should be translated into stable protocol errors.

Examples include:

- media not found
- unsupported/failed decode
- invalid track
- seek unavailable
- backend unavailable
- permission denied

Clients should not need to parse raw mpv error strings.

## 11. Testing strategy

The backend-neutral Player interface enables testing without real video decoding.

```text
JSON-RPC tests
     |
     v
FakePlayer
```

Recommended layers:

1. protocol serialization/deserialization tests
2. JSON-RPC dispatcher tests
3. PlayerService state-transition tests
4. libmpv adapter integration tests
5. Windows/Android playback tests with representative media

Representative playback fixtures should eventually cover:

- H.264 MP4
- H.264 MKV
- HEVC Main10 MKV
- AV1
- multiple audio tracks
- SRT subtitle
- ASS subtitle
- high-bitrate 4K media

## 12. Initial milestone

The first vertical slice should stay deliberately small:

```text
start service
 -> connect via WebSocket
 -> media.open
 -> player.play / player.pause
 -> player.seek
 -> player.state
 -> event.position
 -> event.playbackState
```

Once this path is stable, add tracks, subtitles, playlist management, diagnostics, and remote-control features.
