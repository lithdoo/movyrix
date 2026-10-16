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

The built-in application UI should connect through this same WebSocket endpoint instead of using a separate privileged control API.

Movyrix may have multiple simultaneous clients:

```text
                    Movyrix Engine
                          |
             +------------+------------+
             |            |            |
        Built-in UI   Android remote   Web client
```

## 2. Multi-client semantics

The Movyrix Engine is the authoritative state owner.

Commands are sent by one client, but confirmed state changes are normally broadcast to all connected clients.

Example:

```text
Client B -> player.pause -> Engine
                         -> playback becomes paused
                         -> event.playbackState -> Client A
                                                -> Client B
                                                -> Client C
```

Clients should not assume that a request succeeded until the response and/or resulting state event confirms it.

On connection or reconnection, the expected synchronization flow is:

```text
connect
  -> client.hello
  -> player.state
  -> render snapshot
  -> consume incremental events
```

This is the core synchronization model:

```text
Snapshot + Incremental Events
```

## 3. Client identification

### `client.hello`

A client should identify itself immediately after establishing the WebSocket connection.

Example request:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "client.hello",
  "params": {
    "name": "Movyrix Android",
    "version": "0.1.0",
    "protocolVersion": "0.1",
    "kind": "built-in",
    "platform": "android"
  }
}
```

Suggested `kind` values:

```text
built-in
remote
web
automation
```

Example response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "clientId": "c_01JABC...",
    "serverVersion": "0.1.0",
    "protocolVersion": "0.1",
    "authenticated": true
  }
}
```

`clientId` is an opaque server-assigned identifier for the connection/client session.

Future versions may include capabilities, permissions, authentication state, device identity, or negotiated protocol features.

## 4. Request

```json
{
  "jsonrpc": "2.0",
  "id": 10,
  "method": "player.pause"
}
```

A command that accepts parameters:

```json
{
  "jsonrpc": "2.0",
  "id": 11,
  "method": "player.seek",
  "params": {
    "position": 123.5
  }
}
```

## 5. Successful response

```json
{
  "jsonrpc": "2.0",
  "id": 11,
  "result": {
    "ok": true
  }
}
```

Read methods may return structured data directly:

```json
{
  "jsonrpc": "2.0",
  "id": 12,
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

## 6. Error response

```json
{
  "jsonrpc": "2.0",
  "id": 13,
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
-32009  Authentication required
-32010  Unauthorized client
-32011  Protocol version mismatch
```

Standard JSON-RPC errors should retain their standard meanings.

## 7. Notifications and events

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

Unless explicitly documented otherwise, state-changing domain events should be broadcast to every authorized connected client.

## 8. Player methods

### `player.play`

Resume/start playback.

```json
{
  "jsonrpc": "2.0",
  "id": 20,
  "method": "player.play"
}
```

### `player.pause`

Pause playback.

```json
{
  "jsonrpc": "2.0",
  "id": 21,
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
  "id": 22,
  "method": "player.seek",
  "params": {
    "position": 600.25
  }
}
```

Future versions may add explicit relative seeking instead of overloading this method.

A completed seek should produce an immediate authoritative position update for all clients.

### `player.state`

Return a full state snapshot. This is the primary connection/reconnection synchronization method.

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

## 9. Media methods

### `media.open`

Open a media URI.

```json
{
  "jsonrpc": "2.0",
  "id": 30,
  "method": "media.open",
  "params": {
    "uri": "D:\\Movies\\movie.mkv"
  }
}
```

Opening a media item does not require clients to know backend-specific file-loading commands.

Remote clients may be subject to stricter URI/file-access policy than the built-in client. The protocol should not imply unrestricted remote filesystem access.

### `media.close`

Close the current media.

### `media.info`

Return metadata for the currently loaded media.

## 10. Audio methods

### `audio.volume.set`

```json
{
  "jsonrpc": "2.0",
  "id": 40,
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
  "id": 41,
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
  "id": 42,
  "method": "audio.track.set",
  "params": {
    "id": "audio:2"
  }
}
```

## 11. Subtitle methods

### `subtitle.track.set`

```json
{
  "jsonrpc": "2.0",
  "id": 50,
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
  "id": 51,
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
  "id": 52,
  "method": "subtitle.add",
  "params": {
    "uri": "D:\\Movies\\movie.zh-CN.ass"
  }
}
```

### `subtitle.delay.set`

Set subtitle timing offset in seconds.

## 12. Track discovery

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

## 13. Playlist methods

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

## 14. System methods

Planned methods include:

```text
system.info
system.version
system.capabilities
```

`system.capabilities` should eventually expose feature support without leaking backend implementation details.

## 15. Event catalog

Initial planned notifications:

```text
event.connected
event.clientConnected
event.clientDisconnected
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

Client presence events may later be restricted to privileged/built-in clients if exposing connected-client metadata would create a privacy concern.

## 16. Position update policy

The native backend may update playback position far more frequently than clients need.

Recommended server behavior:

- emit `event.position` around 4-10 Hz during normal playback
- emit an immediate position event after a completed seek
- broadcast the authoritative position to all relevant clients
- allow clients to interpolate the progress bar between notifications

This keeps UI smooth without generating unnecessary JSON/WebSocket traffic.

## 17. Connection lifecycle

Recommended initial client lifecycle:

```text
1. Open WebSocket
2. Send client.hello
3. Complete authentication/pairing if required
4. Call player.state
5. Render snapshot
6. Consume events
7. Reconnect on transport failure
8. Repeat state snapshot after reconnect
```

A client must assume that events may have been missed while disconnected. Reconnection therefore requires a fresh snapshot rather than replaying assumptions from local state.

## 18. Versioning

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

`client.hello` should eventually participate in protocol-version negotiation or reject incompatible clients explicitly.

## 19. Local vs. remote security

### Local-only mode

By default:

```text
bind: 127.0.0.1 / ::1
```

The built-in UI connects over loopback.

Baseline requirements:

- validate browser origins
- validate request schema and parameter ranges
- cap message size
- limit concurrent connections
- reject arbitrary backend commands/properties
- avoid generic filesystem RPC methods

### Remote-control mode

LAN access must be explicitly enabled by the user before binding to a non-loopback interface.

Remote access should require a pairing/authentication mechanism such as:

```text
remote client connects
    -> pairing required
    -> user confirms code
    -> server issues credential/token
    -> subsequent authenticated connection
```

The server should support client credential revocation.

Remote clients may receive a smaller permission set than the built-in client, particularly for filesystem and application-management operations.

## 20. Built-in client principle

The built-in application UI should use the same RPC methods and event semantics as external clients.

Benefits:

- the primary UI continuously exercises the public protocol
- fewer hidden internal-only APIs
- consistent local and remote behavior
- easier integration testing
- easier future separation of Engine and UI processes

An optimization layer may later replace loopback WebSocket with an in-process transport, but only if it preserves the exact same protocol semantics.

## 21. Non-goal

The RPC channel is **not** a media transport.

Do not send decoded video frames, PCM audio, or large media buffers through JSON-RPC/WebSocket. Media rendering remains in the native playback pipeline.
