# Movyrix

Movyrix is a cross-platform local media player project focused on broad format support, native decoding, and a clean control protocol.

The current target platforms are **Windows** and **Android**. The UI and the playback engine are intentionally decoupled: every UI acts as a Movyrix client and controls the engine through **WebSocket + JSON-RPC 2.0**, while video/audio decoding and rendering remain in the native media pipeline.

## Goals

- Play common local media containers such as MP4, MKV, AVI, FLV, and MOV.
- Support codecs such as H.264, H.265/HEVC, AV1, VP9, AAC, FLAC, AC3, DTS, and others supported by the playback backend.
- Support multiple audio tracks and subtitle tracks.
- Support external subtitles such as SRT, ASS, and SSA, with room for bitmap subtitle support where available.
- Use platform hardware decoding and GPU rendering whenever possible.
- Keep the UI independent from the playback backend.
- Expose all player controls and state through a stable JSON-RPC protocol.
- Use the same protocol for the built-in application UI and external controllers.
- Support multiple simultaneous clients and keep them synchronized from one authoritative engine state.
- Allow future clients such as desktop UI, Android UI, browser-based remote controls, and LAN remotes to reuse the same protocol.

## Architecture

Movyrix separates the **control plane** from the **media plane**.

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
                      Player Service
                              |
                       State / Event Bus
                              |
                    Playback Backend Adapter
                              |
                            libmpv
                              |
          +-------------------+-------------------+
          |                   |                   |
        FFmpeg              libass        HW Decoder / GPU
```

The built-in application UI is intentionally a normal WebSocket client. It does not receive a privileged control path. This keeps the public RPC API continuously exercised by the primary UI and makes local and remote control behavior consistent.

### Control plane

WebSocket carries only small control and state messages:

```text
play
pause
seek
volume
track selection
metadata
position
playback state
errors
```

Video frames and PCM audio are **not** transported through JSON-RPC/WebSocket.

### Media plane

The playback backend owns decoding, rendering, subtitle composition, and audio output. Heavy media data stays in native/GPU paths.

Typical Windows path:

```text
media file -> FFmpeg/libmpv -> hardware decoder -> GPU -> display
```

Typical Android path:

```text
media file -> FFmpeg/libmpv -> MediaCodec/native surface -> display
```

This design avoids copying raw video frames through Dart, JavaScript, or WebSocket.

## Application model

A Movyrix application may contain both an engine and a built-in client.

### Android

The Android application is expected to be a single user-facing app containing both the video UI and the player service:

```text
Movyrix APK
|
+-- Flutter/UI layer
|    +-- Built-in WebSocket client
|         -> ws://127.0.0.1:<port>/rpc
|
+-- Playback service / Engine
|    +-- WebSocket server
|    +-- JSON-RPC dispatcher
|    +-- State store / event broadcaster
|    +-- libmpv
|
+-- Native video surface
```

The user does not switch between a separate player app and controller app. The normal player UI is itself the built-in RPC client.

### Windows

Windows can use the same single-application model, while still allowing the engine to evolve into a separately hosted process if useful later.

```text
Movyrix Desktop
|
+-- Built-in UI client
+-- Movyrix Engine
+-- libmpv
```

External clients can connect at the same time, for example an Android phone controlling playback on a Windows PC over the LAN.

## Multi-client semantics

Movyrix supports one built-in client and zero or more external clients simultaneously.

A command originates from one client, but resulting state changes are broadcast to every connected client:

```text
Android remote sends player.pause
              |
              v
         Movyrix Engine
              |
       playback becomes paused
              |
       +------+------+------+
       |             |      |
       v             v      v
 Built-in UI     Android   Web
 event.playbackState = paused
```

The **Movyrix Engine is the single source of truth**. Clients must not treat local optimistic UI state as authoritative.

On connection or reconnection, a client should:

```text
connect
  -> identify itself
  -> request player.state
  -> render the full snapshot
  -> consume incremental events
```

This is the core synchronization model:

```text
Snapshot + Incremental Events
```

## JSON-RPC example

Open media:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "media.open",
  "params": {
    "uri": "D:\\Movies\\example.mkv"
  }
}
```

Pause:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "player.pause"
}
```

Seek:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "player.seek",
  "params": {
    "position": 123.5
  }
}
```

Server-side event notification:

```json
{
  "jsonrpc": "2.0",
  "method": "event.position",
  "params": {
    "position": 123.7
  }
}
```

## API namespaces

The protocol is intentionally independent from mpv-specific property names.

Planned namespaces include:

```text
client.*
player.*
media.*
audio.*
video.*
subtitle.*
playlist.*
system.*
event.*
```

Examples:

```text
client.hello

player.play
player.pause
player.stop
player.seek
player.state

media.open
media.close
media.info

audio.volume.set
audio.mute.set
audio.track.set

subtitle.track.set
subtitle.add
subtitle.delay.set

playlist.add
playlist.remove
playlist.next
playlist.previous
```

The public RPC API should not expose raw implementation details such as `mpv.setProperty("sid", ...)`. Backend-specific concepts belong behind an adapter.

## State model

The playback engine is the source of truth.

```text
User action
    |
    v
JSON-RPC command
    |
    v
Player Core changes state
    |
    v
JSON-RPC event broadcast
    |
    v
All clients update UI
```

A newly connected client should first request a complete state snapshot with `player.state`, then consume incremental events.

Position events should generally be emitted at a modest rate (for example 4-10 Hz); clients can interpolate progress visually. Seek operations should be throttled or committed when the user finishes dragging the seek bar.

## Local and remote access

By default, the RPC server should bind only to loopback:

```text
127.0.0.1
::1
```

This is sufficient for the built-in client.

Remote control should be an explicit user-enabled mode. When LAN access is enabled, Movyrix should add pairing/authentication before accepting control connections and should maintain a client registry for diagnostics and authorization.

Conceptually:

```text
Default:
  built-in client -> 127.0.0.1

Remote control enabled:
  local client  -> loopback
  remote client -> LAN address + pairing/token
```

## Initial technology direction

- **Playback:** libmpv / FFmpeg / libass
- **Protocol:** WebSocket + JSON-RPC 2.0
- **Primary platforms:** Windows + Android
- **UI candidate:** Flutter
- **Platform decoding:** native hardware decoding where available

The Player Core and RPC protocol should remain independent from Flutter so other clients can be added later.

## Repository direction

A likely project layout is:

```text
movyrix/
├── apps/
│   ├── desktop/
│   └── android/
├── packages/
│   ├── protocol/
│   └── client/
├── core/
│   ├── player/
│   ├── rpc/
│   └── backends/
└── docs/
    ├── architecture.md
    └── rpc.md
```

The exact language/package layout can evolve as implementation starts; protocol and playback abstractions should remain stable boundaries.

## Design principles

1. **Do not send decoded video frames over WebSocket.**
2. **Keep native media work out of the UI layer.**
3. **Treat the Player Core as the single source of truth.**
4. **Treat the built-in UI as a normal Movyrix client.**
5. **Broadcast confirmed state changes to all connected clients.**
6. **Use notifications for asynchronous state changes instead of polling.**
7. **Keep the public protocol backend-agnostic.**
8. **Prefer hardware decoding and GPU rendering.**
9. **Bind locally by default; require explicit authenticated configuration for LAN control.**

See [`docs/architecture.md`](docs/architecture.md) and [`docs/rpc.md`](docs/rpc.md) for the evolving design.

## Status

Movyrix is currently in the architecture/bootstrap phase. The next milestone is a minimal engine capable of accepting a built-in WebSocket client, opening a local file, playing/pausing, seeking, reporting state, broadcasting state changes, and allowing additional external clients to connect through the same JSON-RPC API.
