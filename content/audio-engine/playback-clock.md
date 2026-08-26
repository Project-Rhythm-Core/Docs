---
title: Playback Clock
tags:
  - audio
  - rust
  - timing
---

`PlaybackClock` (`rust-core/src/audio/clock.rs`) is the authoritative answer to "where
are we in the song". It lives entirely in atomics, so the real-time audio callback can
publish updates without ever taking a lock and any other thread can read the position for
free.

## The wrong way to count

The obvious implementation — count frames as you copy them into the device buffer — is
wrong, and wrong by a lot. When the callback hands over 256 frames, those frames are not
audible yet; they sit in the device queue behind everything already submitted. A cursor
based counter runs *ahead* of reality by the full output latency.

The clock instead stores a single number:

> `origin_nanos` — the instant at which **sample 0 reaches the speaker**.

Position then becomes a subtraction any thread can perform:

```rust
pub fn position_ms(&self, duration_ms: f64) -> f64 {
    if !self.has_anchor.load(Ordering::Acquire) {
        return 0.0;
    }

    let origin = self.origin_nanos.load(Ordering::Relaxed) as i128;
    let offset = self.calibration_offset_nanos.load(Ordering::Relaxed) as i128;
    let elapsed_ms = (now_nanos() as i128 - origin - offset) as f64 / 1_000_000.0;

    elapsed_ms.clamp(0.0, duration_ms)
}
```

All timestamps are measured against a shared `EPOCH` so they fit in a `u64` of
nanoseconds and can live in an atomic.

## Finding the audible instant

Every callback computes how far in the future its first frame will be heard, from two
independent estimates, and takes the larger:

```rust
let reported_lead_nanos = timestamps
    .playback
    .duration_since(timestamps.callback)
    .as_nanos() as u64;

let nominal_drain_nanos = frames_submitted * NANOS_PER_SECOND / sample_rate;
let estimated_lead_nanos = (first_callback_nanos
    + (nominal_drain_nanos as f64 / rate_ratio) as u64)
    .saturating_sub(now);

let lead_nanos = reported_lead_nanos.max(estimated_lead_nanos);
let audible_at_nanos = now + lead_nanos;
```

- **`reported`** is what the backend claims. It is the better number when it is real —
  but ALSA behind PipeWire or PulseAudio frequently reports **zero**, so it cannot be
  trusted alone.
- **`estimated`** comes from first principles: the frames already handed over take
  `frames / (rate × ratio)` to drain from the first callback. Whatever is left is how far
  ahead this buffer sits.

That `ratio` is the crystal correction, and it is not optional — see [[#Hardware rate]].
Converting a frame count with the *nominal* rate makes `estimated` diverge from reality
at exactly the drift rate, growing without bound. It would then win the `max()` forever,
even on backends whose timestamps are perfectly good.

Subtracting the cursor position gives the candidate origin — the instant sample 0 was, or
would have been, audible:

```rust
let cursor_nanos = playback_cursor as u64 * NANOS_PER_SECOND / sample_rate;
let candidate_origin_nanos = audible_at_nanos.saturating_sub(cursor_nanos);
```

## Smoothing

A raw candidate per callback would jitter by a buffer period. `update_anchor` folds it in
with a moving average whose window grows to a cap:

```rust
const ANCHOR_SMOOTHING_WINDOW: u64 = 32;

let divisor = (update_index + 1).min(ANCHOR_SMOOTHING_WINDOW) as i128;
let previous = self.origin_nanos.load(Ordering::Relaxed) as i128;
let smoothed = previous + (candidate_origin_nanos as i128 - previous) / divisor;
```

The `update_index + 1` divisor makes this a **cumulative mean for the first 32 updates** —
converging in a couple of hundred milliseconds — and then an **EMA with α = 1/32**
thereafter, which absorbs per-callback scheduling jitter without visible jumps on screen.

The EMA is also how crystal drift reaches the reported position. Once the lead is
converted with the real rate, the candidate no longer sits still: it moves at
`1 − ratio` per second, so `now − origin` advances at `ratio` — exactly the rate the song
is really being consumed at.

The first update after a start or seek skips the average entirely: it stores the
candidate verbatim, records the output latency snapshot, and sets `has_anchor`.

> [!caution] Do not correct twice
> No scaling is applied when reading the position; the correction is already baked into
> the anchor. Scaling `position_ms` by `ratio` *as well* squares it. It is a subtle
> mistake because it looks right, and only shows up as roughly double the expected error
> in a long measurement run.

## Hardware rate

The sound card's crystal does not run at exactly the nominal rate. The difference is
linear in time: at 100 ppm it is ~24 ms over a four-minute song, comfortably enough to
change a judgement between the first chorus and the last.

Nothing in the buffer arithmetic can recover that number, because the real rate never
enters it — it has to be measured against an independent clock. The callback counts
frames actually delivered per second of monotonic time:

```rust
let window = now.saturating_sub(rate_window_start_nanos);
if window > MIN_RATE_WINDOW_NANOS {
    let delivered = frames_submitted.saturating_sub(rate_window_start_frames) as f64;
    let estimate = delivered * NANOS_PER_SECOND as f64
        / (window as f64 * sample_rate as f64);

    if (estimate - 1.0).abs() < MAX_RATE_DEVIATION {
        rate_ratio = estimate;
        clock.publish_rate_ratio(rate_ratio);
    }
}
```

Three details make it usable, and each of them matters:

- **Differences, not absolutes.** `frames_submitted` runs ahead of what has been consumed
  by the buffer lead. Taking the difference between two instants cancels that lead, which
  would otherwise bias the estimate by `lead / window`.

- **The reference is taken late.** `RATE_SETTLE_NANOS` (2 s) delays it until the device
  has been warm for a while. "Warm" only means the device has *started* queueing ahead; it
  keeps filling for some time after that, and anchoring mid-fill counts that growth as
  frames delivered.

  > [!important] This one is easy to get wrong
  > Anchoring at the first warm callback produced estimates of **1850 ppm** decaying to
  > 380 ppm over 40 s — a bias far larger than the drift being measured, shrinking only
  > as `1/window`. With the settle delay, the same machine reads within a few tens of ppm
  > of zero and stays there. A monotonic decay in this number is the signature of a
  > mis-placed reference, not of a clock settling.

- **The window keeps growing.** It runs to the end of the session, so the estimate's
  noise — callback scheduling jitter divided by the window — shrinks with it.
  `MIN_RATE_WINDOW_NANOS` (10 s) is when it first becomes worth believing, and
  `MAX_RATE_DEVIATION` (1 %) rejects underruns and misbehaving backends rather than
  letting them poison the clock.

`rate_ratio` is exposed as `PlaybackStats.rateRatio`, and stays at exactly `1.0` until
the first estimate lands.

> [!note] On a well-behaved backend it does nothing
> When `reported` is real and larger than the estimate, it wins the `max()` and the rate
> correction never reaches the anchor at all. That is the healthy case: the backend's own
> timestamps already describe reality. The estimate is there for the backends that report
> zero, and is worth watching as a diagnostic either way.

## State

| Field | Written by | Meaning |
| --- | --- | --- |
| `is_armed` | `arm()` / `request_seek()` | Playback requested; the callback copies song data instead of silence |
| `has_anchor` | Audio thread, `request_seek()` | `origin_nanos` holds a real measurement; until then position reads 0 |
| `origin_nanos` | Audio thread | The instant sample 0 reaches the speaker |
| `anchor_updates` | Audio thread | Updates since the last start or seek; drives the smoothing window |
| `is_device_warm` | Audio thread | The device is queueing ahead |
| `pending_seek_frames` | `request_seek()` | Requested position, `-1` = none |
| `rate_ratio_bits` | Audio thread | Real rate over nominal, as `f64` bits |
| `calibration_offset_nanos` | `set_calibration_offset_ms()` | Manual calibration |
| `output_latency_nanos` | Audio thread | Lead on the first armed callback; a snapshot |

`is_device_warm` flips the moment the estimated lead is positive — the first time the
device is genuinely running ahead of the callback rather than starving. See
[[latency-and-calibration]].

## Arming is cheap

`arm()` does not open anything. The stream is already running and emitting silence:

```rust
pub fn arm(&self) {
    self.is_armed.store(true, Ordering::Release);
}
```

Sound therefore begins on the *next* callback — one buffer period — rather than after a
device open, which costs hundreds of milliseconds.

## Seeking and restart

`request_seek` invalidates the anchor, posts the target, and arms, in that order:

```rust
pub fn request_seek(&self, frame: u64) {
    self.has_anchor.store(false, Ordering::Release);
    self.pending_seek_frames.store(/* … */, Ordering::Release);
    self.is_armed.store(true, Ordering::Release);
}
```

The audio thread claims the request with a `swap(-1, AcqRel)` — so it is honoured exactly
once — then moves the cursor and resets the anchor. The next callback re-anchors from
scratch.

> [!important] Clearing `has_anchor` first is load-bearing
> The audio thread only picks the request up on its next callback, up to a buffer period
> later, and an IPC round trip is much shorter than that. Without that first line,
> `position_ms` keeps answering from the *old* anchor, the renderer's wait loop exits
> immediately, and the handshake latches onto a stale origin — leaving the picture out of
> sync until the next resync notices. Clearing it on the calling thread makes position
> read 0 synchronously, so consumers wait for the genuine re-anchor.

> [!tip] Seeking to arbitrary positions
> `pending_seek_frames` is a general frame offset clamped to the track length; only
> `restart()` uses it today, always with `0`. Exposing a `seek(ms)` binding needs nothing
> more than a new `#[napi]` function that stores `ms * rate / 1000`.

See also: [[native-module]], [[renderer-clock]], [[latency-and-calibration]]
