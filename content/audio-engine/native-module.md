---
title: Native Module
tags:
  - audio
  - rust
---

`rust-core` is a Rust crate compiled to a Node addon with [napi-rs](https://napi.rs/) and
loaded by the Electron main process. It owns the decoded PCM, the output device, and the
authoritative playback clock.

```toml
[lib]
crate-type = ["cdylib"]

cpal        = "0.18"
napi        = { version = "3", default-features = false, features = ["napi4"] }
napi-derive = "3"
symphonia   = { version = "0.6", features = ["mp3", "ogg", "wav", "flac"] }
once_cell   = "1"
```

The crate is split so each concern can grow on its own:

| Module | Responsibility |
| --- | --- |
| `audio/decoder.rs` | File → interleaved `f32` PCM |
| `audio/device.rs` | Stream configuration, channel mapping, resampling |
| `audio/clock.rs` | Timing state, entirely in atomics |
| `audio/engine.rs` | The stream and its real-time callback |
| `lib.rs` | N-API bindings only — no logic |

> [!note] The type definitions are generated
> `pnpm run build:native` emits `rust-core/index.d.ts` from the Rust doc comments, so the
> TypeScript surface cannot drift from the Rust one. Do not hand-edit it: the next build
> overwrites it.

## Decoding — `decoder.rs`

`decode_file(path)` reads the entire file into an interleaved `Vec<f32>`:

1. The file extension is fed to Symphonia's probe as a `Hint`, so format detection
   usually succeeds on the first try.
2. `first_track(TrackType::Audio)` selects the track, and its `AudioCodecParameters`
   supply the rate and channel layout.
3. `Vec::with_capacity` is sized from `num_frames` when the container declares it,
   falling back to sixty seconds' worth, which avoids dozens of reallocations mid-decode.
4. One scratch buffer is reused across packets, grown only when a packet needs more room
   than the last.

Error handling is deliberately asymmetric:

| Condition | Behaviour |
| --- | --- |
| `next_packet()` returns `None` | End of stream, keep what was decoded |
| `DecodeError` on one packet | Skip the packet, continue |
| Any other error | Abort the load |
| Zero samples produced | Abort with "the file contains no decodable samples" |

A handful of corrupt packets in the middle of a track should not fail the whole load.

### Encoder delay

Lossy encoders prepend priming samples that are not part of the music, and record how
many in a header. Symphonia trims them, so decoding starts at the first real sample
rather than at the start of the bitstream.

The correction is not small. A typical LAME-encoded MP3 carries 1105 priming frames —
**25 ms at 44.1 kHz**, comfortably inside a rhythm game's "Perfect" window. A decoder
that ignores the header reports a track 25 ms longer and places every sample 25 ms late
relative to a chart authored against the true audio start.

> [!important] Timing data is tied to the decoder
> Because the trim depends on what the encoder wrote, per-song offsets are only valid
> for the decoder that produced them. Changing decoder — or feeding in a file with no
> encoder-delay header, which gets no trim at all — shifts the whole timeline. Verify
> against a known reference (`ffmpeg -f s16le` and a frame count will do) rather than
> assuming two decoders agree, because container duration and decoded frame count
> legitimately differ by exactly this amount.

> [!warning] Everything lives in RAM
> There is no streaming decode. A ten-minute stereo track at 44.1 kHz costs around
> 230 MB of `f32` samples. This is a deliberate trade: it removes disk I/O and decoder
> jitter from the audio callback entirely, and makes seeking exact by construction —
> a position is an index, not a re-decode from the nearest frame boundary. Loading many
> charts back to back means unloading each before the next.

## Threading — `lib.rs`

Decoding a whole file blocks for hundreds of milliseconds, which Electron's main thread
cannot afford. `LoadAudioTask` implements napi's `Task` trait to split the work:

```rust
impl Task for LoadAudioTask {
    // Runs on a libuv worker thread.
    fn compute(&mut self) -> Result<DecodedAudio> {
        audio::decode_file(&self.path)
    }

    // Back on the main thread, where cpal::Stream must live.
    fn resolve(&mut self, _env: Env, decoded: DecodedAudio) -> Result<AudioInfo> {
        let engine = AudioEngine::prepare(decoded)?;
        /* … */
    }
}
```

The engine itself is a process-wide singleton:

```rust
static ENGINE: Lazy<Mutex<Option<AudioEngine>>> = Lazy::new(|| Mutex::new(None));
```

`cpal::Stream` is not `Send` on every backend, so `AudioEngine` carries an
`unsafe impl Send`. This holds only because the stream is created and dropped exclusively
from Node's main thread — the one thread that touches the mutex.

> [!caution] One track at a time
> Loading again replaces what was there, and `unload()` drops the engine, releasing the
> device. Previews, crossfades and hit sounds all need more than one voice at once, which
> means growing this into a mixer with a handle per voice — at which point the global
> becomes a registry and the `unsafe impl Send` can be replaced by a dedicated audio
> thread with a command channel.

## Device negotiation — `device.rs`

`negotiate` adapts the decoded PCM to something the device will accept and works out the
stream configuration. Everything here runs once, at load, off the real-time path.

### Sample format

The stream is built as `f32`, and cpal panics *inside the callback* if the host hands
back a different sample type. The device's supported configurations are checked up front
so an unsupported device fails at load with a message that says why:

```
the output device does not support 32-bit float samples
```

Supporting `i16`/`u16` output means making the callback generic over the format.

### Buffer size

```rust
const PREFERRED_BUFFER_FRAMES: u32 = 256;
```

256 frames is ~5.3 ms at 48 kHz, ~5.8 ms at 44.1 kHz. The system default is typically
1024-8192 frames (20-170 ms), which on its own would exceed a rhythm game's entire
timing window. The trade is CPU headroom: smaller buffers mean more callbacks and less
slack before an underrun.

`pick_buffer_size` walks the supported configurations, finds one matching the channel
count and covering the sample rate, and clamps 256 into its `min..max` range. If the
device reports no range, it falls back to the backend's default and `bufferFrames` is
reported as `0`.

### Channels and rate

Two fixes are applied in order:

1. **Channels.** If no supported configuration matches the file's channel count,
   `remap_channels` converts to the device default.
2. **Sample rate.** If the rate falls outside every supported range for that channel
   count, `resample_linear` converts to the device rate.

`remap_channels` is a positional fallback, and deliberately conservative in both
directions:

- **Upmixing** copies each source channel to its own position and leaves the rest
  *silent*. Mono is the one special case, feeding both front channels. Nothing is
  invented — a stereo source never puts a copy of one channel into the centre or LFE.
- **Downmixing** folds the extra channels onto the ones that remain, so front-left
  content stays left, and averages so the result cannot clip. No channel is dropped.

> [!caution] Neither conversion is production grade
> This is not a downmix matrix — ITU-R BS.775 for 5.1 to stereo, and an explicit LFE
> policy, are the natural next step once surround output matters. `resample_linear` is
> plain linear interpolation: cheap and adequate while assets ship at the device's rate,
> but it will alias audibly on large ratio changes, so accepting arbitrary user-supplied
> audio means swapping in a windowed-sinc or polyphase implementation.

Both functions are pure, and covered by unit tests (`pnpm run test:native`).

The values actually negotiated are reported back in `AudioInfo`, which is why
`sampleRate` and `channels` may differ from the source file.

## The audio callback — `engine.rs`

The callback is the only place that touches the sample buffer, and it holds its state as
plain locals across invocations — nothing is shared, so nothing needs to be atomic:

| Variable | Meaning |
| --- | --- |
| `first_callback_nanos` | When the hardware began consuming |
| `frames_submitted` | Frames handed to the device, silence included |
| `playback_cursor` | Frames of the song delivered so far |
| `rate_window_start_*` | Reference the hardware rate is measured against |
| `rate_ratio` | Real rate over nominal; a local copy of what is published |

Per invocation it timestamps itself, updates the rate estimate, works out how far ahead
this buffer will be audible, services any pending seek, and then either copies song
samples or fills with zeros. The timing details are in [[playback-clock]].

Copying is bounds-safe by construction — the tail past the end of the song is
zero-filled:

```rust
let start = playback_cursor * channels;
let available = samples.len().saturating_sub(start).min(output.len());
output[..available].copy_from_slice(&samples[start..start + available]);
output[available..].fill(0.0);
```

> [!note] Playback does not stop by itself
> Once the cursor reaches the end the callback emits silence forever, but the clock stays
> armed and position clamps to the duration. There is no end-of-track event; callers
> compare position against duration themselves.

See also: [[playback-clock]], [[api-reference]]
