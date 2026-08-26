---
title: API Reference
tags:
  - audio
  - reference
---

The same surface appears three times — as Rust `#[napi]` exports, as `ipcMain` channels,
and as `window.electronAPI.audio` in the renderer. This page lists all three plus the
data types.

> [!note] The generated definitions are authoritative
> `rust-core/index.d.ts` is emitted from the Rust doc comments by `pnpm run build:native`
> and cannot drift from the implementation. `src/lib/audio/types.ts` restates the same
> shapes for the renderer, which cannot import across the process boundary.

## Data types

### `AudioInfo`

Returned by `loadAudio()`. Describes the stream **as actually opened**, which may differ
from the source file if channels were remapped or the rate resampled.

| Field | Type | Meaning |
| --- | --- | --- |
| `durationMs` | `number` | Track length, from the decoded frame count |
| `sampleRate` | `number` | Rate the device was opened at |
| `channels` | `number` | Channel count the device was opened with |
| `deviceName` | `string` | Device name, or `"unknown"` if unavailable |
| `bufferFrames` | `number` | Frames per callback; `0` = backend chose |
| `bufferMs` | `number` | `bufferFrames / sampleRate × 1000`; `0` when `bufferFrames` is `0` |

### `PlaybackStats`

| Field | Type | Meaning |
| --- | --- | --- |
| `positionMs` | `number` | Current audible position, calibration applied |
| `durationMs` | `number` | Track length |
| `outputLatencyMs` | `number` | Lead measured on the first armed callback (snapshot) |
| `offsetMs` | `number` | Manual calibration currently applied |
| `rateRatio` | `number` | Real hardware rate over nominal; `1.0` until measured (diagnostic) |
| `playing` | `boolean` | Playback has been armed |
| `ready` | `boolean` | The device is queueing ahead |

## Functions

| Native (`rust-core`) | IPC channel | Renderer (`electronAPI.audio`) |
| --- | --- | --- |
| `loadAudio(path)` | `audio:load` | `load(filePath)` |
| `play()` | `audio:play` | `play()` |
| `restart()` | `audio:restart` | `restart()` |
| `unload()` | `audio:unload` | `unload()` |
| `isReady()` | `audio:is-ready` | `isReady()` |
| `setOffsetMs(ms)` | `audio:set-offset-ms` | `setOffsetMs(offsetMs)` |
| `getPositionMs()` | `audio:position` | `position()` |
| `getStats()` | `audio:stats` | `stats()` |

Channel names live in `electron/audio-channels.cjs`, which is kept dependency-free so the
preload script can require it. Everything on `electronAPI.audio` returns a `Promise`,
including the void operations, because it all crosses `ipcRenderer.invoke`.

### `loadAudio(path): Promise<AudioInfo>`

Decodes the file on a libuv worker, then opens the device on the main thread and starts
the stream emitting silence. Replaces any previously loaded track. Rejects on unreadable
files, unrecognised formats, missing audio tracks, devices without `f32` support, or when
no output device exists.

The path is resolved by the caller; the main process passes it straight through.

### `play(): void`

Arms playback. Sound begins on the next callback. Throws if nothing is loaded. Calling it
while already playing has no effect — use `restart()` to go back to the beginning.

### `restart(): void`

Seeks to frame 0 and arms, without reopening the device. Instantaneous. Throws if nothing
is loaded. Invalidates the anchor synchronously, so `getPositionMs()` reports `0` from the
moment it returns — see [[playback-clock#Seeking and restart]].

### `getPositionMs(): number`

Current audible position in milliseconds, clamped to `[0, durationMs]`. Returns `0` until
the first armed callback has anchored the clock. Cheap by design — an atomic load and a
subtraction — so it is safe to poll.

### `getStats(): PlaybackStats`

Full snapshot. Returns zeroed values (with `rateRatio` at `1.0`) when nothing is loaded,
rather than throwing.

### `isReady(): boolean`

`true` once the device is queueing audio ahead, meaning `play()` will be heard
immediately. `false` before that, and when nothing is loaded.

### `setOffsetMs(offsetMs: number): void`

Manual calibration. Positive means the audio is heard *later* than the clock reports, so
reported position is pulled back. No-op when nothing is loaded. See
[[latency-and-calibration]].

### `unload(): void`

Drops the engine, closing the stream and releasing the device. Always safe to call.
**A subsequent `play()` will throw** — the track has to be loaded again.

## Renderer wrapper

[[renderer-clock|`AudioClock`]] wraps the bridge and is what application code should use:

| Member | Notes |
| --- | --- |
| `load(filePath)` | Stores the result in `.info` |
| `waitUntilReady(timeoutMs = 5000)` | Polls `isReady()` every 20 ms; returns `false` on timeout |
| `play()` / `restart()` | Call through, then latch the local clock |
| `unload()` | Clears the resync interval, then releases the device |
| `setOffsetMs(offsetMs)` | Passes calibration through to the engine |
| `positionMs()` | Local, no IPC — call every frame |
| `isRunning` | Whether the local clock is latched |
| `info`, `stats`, `syncErrorMs`, `roundTripMs` | Diagnostics |

`play()` and `restart()` throw if the audio has not become audible within 3 s.

## Error messages

Native errors surface as `Error` with `Status::GenericFailure`.

| Message | Cause |
| --- | --- |
| `no audio loaded: call loadAudio() first` | `play()` / `restart()` with no engine |
| `could not open '<path>': …` | File open failed |
| `unrecognised audio format: …` | The probe could not identify the container |
| `the file contains no audio track` | No audio track in the container |
| `the audio track declares no codec parameters` | Track present but unplayable |
| `unknown sample rate` / `unknown channel layout` | Codec parameters incomplete |
| `could not create a decoder: …` | No registered decoder for the codec |
| `error reading the file: …` | Demuxer failure part-way through |
| `error decoding: …` | Unrecoverable decoder failure |
| `the file contains no decodable samples` | Decode produced nothing |
| `no output device available` | No default output device |
| `could not read the default output configuration: …` | Device query failed |
| `could not list device configurations: …` | Device query failed |
| `the output device does not support 32-bit float samples` | Device format unsupported |
| `could not build the output stream: …` | Stream creation failed |
| `could not start the stream: …` | Stream start failed |

See also: [[native-module]], [[renderer-clock]], [[latency-and-calibration]]
