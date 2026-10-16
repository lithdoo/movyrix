# Movyrix Architecture

## 1. Overview

Movyrix is designed around a strict separation between the user interface, the player control protocol, and the native playback engine.

```text
                         Movyrix Engine
                    WebSocket + JSON-RPC 2.0
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
      Built-in Client   External Client   External Client
        App UI             Phone              Web
             \                |                /
              +---------------+---------------+
                              |
                    Connection Manager
                              |
                    JSON-RPC Dispatcher
                              |
                       Player Service
                              |
                    State Store / Event Bus
                              |
                  Playback Backend Adapter
                              |
                            libmpv
                              |
                 Native Media Pipeline
```

The important boundary is that RPC transports **commands and state**, not decoded media frames.

The built-in UI deliberately uses the same WebSocket/JSON-RPC interface as external clients. It is not given a separate privileged control API.

## 2. Engine, client, and app

Movyrix distinguishes three concepts:

- **Movyrix Engine**: owns playback state, RPC server, backend adapter, and native media pipeline.
- **Movyrix Client**: any UI or controller connected to the Engine through JSON-RPC.
- **Movyrix App**: a packaged product that may contain both an Engine and a built-in Client.

This allows the same protocol to cover local playback UI, remote control, browser control, and future automation clients.

### Android application model

Android should present a single foreground player application to the user. The app contains both the built-in UI and the player service:

```text
Movyrix APK
|
+-- Activity / Flutter UI
|    +-- Built-in WebSocket Client
|         -> ws://127.0.0.1:<port>/rpc
|
+-- Playback Service / Movyrix Engine
|    +-- WebSocket Server
|    +-- JSON-RPC Dispatcher
|    +-- Player Service
|    +-- State Store / Event Bus
|    +-- libmpv Adapter
|
+-- Native Video Surface
```

The user therefore watches and controls playback in one app. The protocol separation is architectural, not a requirement to split the product into two Android apps.

### Windows application model

Windows may use the same packaged model:

```text
Movyrix Desktop
|
+-- Built-in Client UI
+-- Movyrix Engine
+-- libmpv
```

The Engine can later be hosted in a separate process without changing the public client protocol.

## 3. Control plane vs. media plane

### Control plane

The control plane uses WebSocket + JSON-RPC 2.0 and carries small messages such as:

- client identification
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

## 4. Platform rendering model

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

## 5. Player abstraction

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

## 6. State ownership

The Movyrix Engine is the single source of truth.

Clients should not independently assume that a requested operation succeeded.

Preferred flow:

```text
Client A presses Pause
       |
       v
player.pause request
       |
       v
Player Core pauses playback
       |
       v
Engine state becomes paused
       |
       v
broadcast event.playbackState
       |
       +----------+----------+
       v          v          v
    Client A   Client B   Client C
```

All connected clients receive the confirmed state change, regardless of which client initiated the command.

## 7. Multi-client synchronization

Movyrix must support one built-in client and zero or more external clients simultaneously.

A newly connected or reconnected client should follow this sequence:

```text
connect
  -> client.hello
  -> player.state
  -> render full snapshot
  -> subscribe/consume incremental events
```

The synchronization model is:

```text
Snapshot + Incremental Events
```

The Engine owns sequencing and publication of state changes. Clients are projections of Engine state.

For example, if an Android phone pauses a Windows player, the Windows built-in UI, Android phone, and any browser controller should all converge on `paused` through the same server event.

## 8. Client registry

The Connection Manager should track connected clients for diagnostics, authorization, and future per-client capabilities.

A client may identify itself with fields such as:

```json
{
  "name": "Movyrix Android",
  "version": "0.1.0",
  "protocolVersion": "0.1",
  "kind": "built-in"
}
```

The server may assign an opaque client ID:

```text
c_01J...
```

Possible future client metadata:

- built-in vs. remote
- platform
- application version
- protocol version
- authenticated identity
- permissions/capabilities
- connection time
- remote address

## 9. Event flow

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
Event Broadcaster
        |
        +---- Client A
        +---- Client B
        +---- Client C
```

The service may coalesce high-frequency events. For example, playback position normally only needs to be sent around 4-10 times per second.

## 10. Seeking

Dragging a seek bar can generate hundreds of pointer events. The protocol should not blindly translate every pointer movement into a native seek.

Recommended behaviors:

- Default: update the client slider locally and issue one `player.seek` when dragging ends.
- Preview mode: throttle seeks to a bounded interval (for example 50-100 ms) and discard stale seek intents.

The latest user intent should win.

With multiple clients, a seek should generate a confirmed position/state event that is broadcast to all clients.

## 11. Track model

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

## 12. Transport strategy

The first implementation should use a real loopback WebSocket connection for the built-in client:

```text
Built-in UI
    |
    v
ws://127.0.0.1:<port>/rpc
    |
    v
Movyrix Engine
```

This intentionally exercises the same transport and protocol used by external clients.

If profiling later shows a meaningful reason to optimize local transport, the client layer may introduce a transport abstraction such as:

```text
RpcTransport
+-- WebSocketTransport
+-- InProcessTransport
```

The JSON-RPC method semantics and state model should remain unchanged.

## 13. Security model

The initial server should bind to loopback by default:

```text
127.0.0.1
::1
```

This is the normal mode for the built-in client.

Even on localhost, the service should be designed with browser-origin attacks in mind.

Recommended baseline:

- validate `Origin` for browser clients
- avoid exposing arbitrary filesystem operations
- validate all RPC parameters
- limit message size
- limit concurrent connections
- reject unknown methods
- never execute arbitrary mpv commands supplied by a client

LAN access must be explicit. Before binding to a non-loopback interface, add:

- user-controlled `Allow remote control` setting
- pairing flow or authentication token
- client registry and revocation
- authorization checks
- explicit bind-address configuration

A conceptual pairing flow:

```text
Remote client connects
       |
       v
Engine requires pairing
       |
       v
User confirms code/token
       |
       v
Engine issues client credential
       |
       v
Future authenticated connections
```

## 14. Failure handling

Backend errors should be translated into stable protocol errors.

Examples include:

- media not found
- unsupported/failed decode
- invalid track
- seek unavailable
- backend unavailable
- permission denied
- authentication required
- unauthorized client

Clients should not need to parse raw mpv error strings.

## 15. Testing strategy

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
3. multi-client event broadcast tests
4. reconnect + snapshot synchronization tests
5. PlayerService state-transition tests
6. libmpv adapter integration tests
7. Windows/Android playback tests with representative media
8. authentication/pairing tests before LAN control ships

Representative playback fixtures should eventually cover:

- H.264 MP4
- H.264 MKV
- HEVC Main10 MKV
- AV1
- multiple audio tracks
- SRT subtitle
- ASS subtitle
- high-bitrate 4K media

## 16. Initial milestone

The first vertical slice should stay deliberately small:

```text
start Engine
 -> connect built-in WebSocket client
 -> client.hello
 -> player.state
 -> media.open
 -> player.play / player.pause
 -> player.seek
 -> event.position
 -> event.playbackState broadcast
 -> connect a second test client
 -> verify both clients remain synchronized
```

Once this path is stable, add tracks, subtitles, playlist management, diagnostics, pairing/authentication, and remote-control features.
