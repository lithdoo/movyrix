# Movyrix

Movyrix is a cross-platform local media player project focused on broad format support, native decoding, and a clean remote-control protocol.

The current target platforms are **Windows** and **Android**. The UI and the playback engine are intentionally decoupled: clients control the player through **WebSocket + JSON-RPC 2.0**, while video/audio decoding and rendering remain in the native media pipeline.

## Goals

- Play common local media containers such as MP4, MKV, AVI, FLV, and MOV.
- Support codecs such as H.264, H.265/HEVC, AV1, VP9, AAC, FLAC, AC3, DTS, and others supported by the playback backend.
- Support multiple audio tracks and subtitle tracks.
- Support external subtitles such as SRT, ASS, and SSA, with room for bitmap subtitle support where available.
- Use platform hardware decoding and GPU rendering whenever possible.
- Keep the UI independent from the playback backend.
- Expose all player controls and state through a stable JSON-RPC protocol.
- Allow future clients such as desktop UI, Android UI, browser-based remote controls, and LAN remotes to reuse the same protocol.

## Architecture

Movyrix separates the **control plane** from the **media plane**.

```text
                 Client
        Flutter / Web / Remote
                    |
                    | WebSocket
                    | JSON-RPC 2.0
                    v
+-------------------------------------+
| Movyrix Player Service              |
|                                     |
|  Connection Manager                 |
|          |                          |
|  JSON-RPC Dispatcher                |
|          |                          |
|  Player Service                     |
|          |                          |
|  State / Event Bus                  |
|          |                          |
|  Playback Backend Adapter           |
+----------+--------------------------+
           |
           v
        libmpv
           |
           +-- FFmpeg
           +-- libass
           +-- Hardware Decoder
           +-- GPU Renderer
           +-- Audio Output
```

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
JSON-RPC event
    |
    v
Client updates UI
```

A newly connected client should first request a complete state snapshot with `player.state`, then consume incremental events.

Position events should generally be emitted at a modest rate (for example 4-10 Hz); clients can interpolate progress visually. Seek operations should be throttled or committed when the user finishes dragging the seek bar.

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
4. **Use notifications for asynchronous state changes instead of polling.**
5. **Keep the public protocol backend-agnostic.**
6. **Prefer hardware decoding and GPU rendering.**
7. **Design localhost RPC so it can later support authenticated LAN remote control.**

See [`docs/architecture.md`](docs/architecture.md) and [`docs/rpc.md`](docs/rpc.md) for the evolving design.

## Status

Movyrix is currently in the architecture/bootstrap phase. The next milestone is a minimal player service capable of opening a local file, playing/pausing, seeking, reporting state, and exposing those operations through JSON-RPC over WebSocket.
