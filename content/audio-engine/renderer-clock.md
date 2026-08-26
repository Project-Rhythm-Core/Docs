---
title: Renderer Clock
tags:
  - audio
  - timing
  - renderer
---

`AudioClock` (`src/lib/audio/audio-clock.ts`) is the renderer's view of playback
position. The authoritative clock lives in the native audio thread — but reaching it
costs an IPC round trip, and an `await` inside the render loop means the frame is always
drawn one frame late.

So `AudioClock` does not ask. It estimates, once, the `performance.now()` value at which
the song was at position 0, and from then on position is local arithmetic:

```ts
positionMs(): number {
  if (!this.running) return 0;

  const now = performance.now();
  this.applySlew(now);

  const position = now - this.originMs;
  const duration = this.info?.durationMs ?? Infinity;

  return Math.min(Math.max(position, 0), duration);
}
```

No IPC, no `await`, no allocation. Safe to call every frame from the render loop.

## Tuning constants

```ts
const HANDSHAKE_PROBE_COUNT     = 8;     // probes taken when latching on
const RESYNC_PROBE_COUNT        = 3;     // probes per background re-comparison
const RESYNC_INTERVAL_MS        = 500;   // re-comparison period
const MAX_SLEW_MS_PER_SECOND    = 20;    // fastest allowed correction
const SNAP_THRESHOLD_MS         = 80;    // above this, jump instead of slew
const PLAYBACK_START_TIMEOUT_MS = 3000;  // give up waiting for audible audio
const READY_POLL_INTERVAL_MS    = 20;    // while waiting for the device
```

Every call goes through `window.electronAPI.audio`, the surface the preload script
exposes. `AudioClock` is the only thing in the renderer that should touch it directly.

## The NTP-style handshake

`probe(count)` samples the native clock `count` times and keeps the sample with the
**lowest round-trip time**:

```ts
const sentAt = performance.now();
const positionMs = await this.bridge.position();
const receivedAt = performance.now();

const roundTripMs = receivedAt - sentAt;
if (roundTripMs < best.roundTripMs) {
  best = {
    roundTripMs,
    originMs: (sentAt + receivedAt) / 2 - positionMs,
  };
}
```

The reasoning is the same as NTP's: the reply corresponds to some instant between `t0`
and `t1`. The midpoint is the best single guess, and its error is bounded by `rtt / 2`.
The fastest exchange is therefore the least uncertain, so taking the minimum over eight
probes discards IPC scheduling noise rather than averaging it in.

`origin` — the `performance.now()` at which position was 0 — is `midpoint − position`.

## Latching on

`beginTracking()` runs after every `play()` and `restart()`:

```
play()/restart()
   │
   ├─ poll position() until > 0   (rAF-paced, PLAYBACK_START_TIMEOUT_MS budget)
   │     └─ native returns 0 until sample 0 is genuinely audible
   │
   ├─ probe(8) → keep lowest-RTT sample
   ├─ origin = targetOrigin = best.origin
   └─ start 500 ms resync interval
```

The initial poll matters: right after `play()` the native side still reports `0` because
the anchor has not been built yet ([[playback-clock]]). Latching during that window would
produce an origin that is ahead of reality by the whole warm-up. Polling is paced with
`requestAnimationFrame` so it does not spin, and gives up after 3 s with a thrown error.

## Staying in sync

Every 500 ms, `resync()` takes a smaller `probe(3)` in the background:

```ts
// Dominated by IPC jitter: correcting with it is worse than not correcting.
if (probe.roundTripMs > Math.max(this.roundTripMs * 4, 8)) return;

this.syncErrorMs = probe.originMs - this.originMs;

if (Math.abs(this.syncErrorMs) > SNAP_THRESHOLD_MS) {
  this.originMs = probe.originMs;        // too large to be drift: follow it
  this.targetOriginMs = probe.originMs;
} else {
  this.targetOriginMs = probe.originMs;  // slew towards it
}
```

Three behaviours, in order of aggressiveness:

| Condition | Action | Why |
| --- | --- | --- |
| RTT > 4× the handshake RTT (min 8 ms) | Discard the sample | The reading is dominated by IPC jitter; correcting with it is worse than not correcting |
| \|error\| > 80 ms | Snap immediately | Something real happened — a restart or seek — not drift |
| Otherwise | Move `targetOrigin`, let slew do the work | Sub-threshold drift should never be visible |

`resync()` also refreshes `stats`, which is why `PlaybackStats` is available to the
renderer without any extra call.

## Slewing

`applySlew` is called from `positionMs()`, so it runs once per frame and never on its
own timer:

```ts
const elapsedSeconds = (now - this.lastSlewAtMs) / 1000;
this.lastSlewAtMs = now;

const remaining = this.targetOriginMs - this.originMs;
if (remaining === 0) return;

const step = Math.min(Math.abs(remaining), MAX_SLEW_MS_PER_SECOND * elapsedSeconds);
this.originMs += Math.sign(remaining) * step;
```

At 20 ms of correction per second, a 10 ms error takes half a second to absorb and moves
the origin by 0.33 ms per 60 Hz frame — well below anything a player can perceive. The
alternative, snapping, would make notes visibly stutter every half second.

> [!note] `origin` moves, position does not jump
> Because the correction is applied to the *origin* rather than to the reported position,
> playback position stays monotonic throughout a slew. It simply advances at very
> slightly more or less than 1 ms per ms until the error is gone.

## Lifecycle

```ts
const clock = new AudioClock();

await clock.load(chartAudioPath);  // decode + open device
await clock.waitUntilReady();      // poll isReady(), 20 ms interval, 5 s default budget
await clock.play();                // arm + latch
clock.positionMs();                // every frame, free

await clock.restart();             // back to 0, re-latch, no device reopen
await clock.unload();              // clears the interval, releases the device
```

`load()` and `waitUntilReady()` belong as early as the application can put them — on
screen entry, well before the player can trigger playback — not next to the `play()`
call. See [[latency-and-calibration]] for why that ordering is the whole point.

> [!warning] `unload()` is terminal
> It calls through to the native `unload()`, which drops the engine singleton and
> releases the device. The instance is not reusable afterwards without another `load()`.

## Charts with no audio to follow

Everything downstream of the clock — where notes are drawn, when they are judged, when
they are written off — is a function of one number. A chart with no clock therefore does
not play badly; it does not play at all. Notes sit motionless, and only the handful
already on screen are reachable.

That is exactly what a fully keysounded chart is: `AudioFilename: virtual`, no background
track, and nothing for the clock to anchor to. So the clock is an interface with two
implementations, and gameplay does not know which one it has:

| Clock | Used when | Anchored to |
| --- | --- | --- |
| `AudioClock` | The chart has a background track | The audible position of the audio |
| `SystemClock` | It does not | `performance.now()`, free-running |

> [!caution] The fallback is a stopgap, not a design
> Nothing keeps a free-running clock honest. It is correct only because there is no sound
> to be out of step with — the moment the engine can mix a sample bank, a keysounded chart
> has to be driven by the audio device like any other, because its samples have to land
> where its notes say they do. Until then such charts are playable and silent, and the
> overlay says so rather than pretending otherwise.

## Exposed diagnostics

| Field | Meaning |
| --- | --- |
| `info` | `AudioInfo` from the last `load()` — device, rate, buffer size |
| `stats` | `PlaybackStats` refreshed by the 500 ms resync |
| `syncErrorMs` | Last observed difference between local and native clock |
| `roundTripMs` | Round trip of the accepted handshake probe |
| `isRunning` | Whether the clock is latched |

`syncErrorMs` and `roundTripMs` are the two worth watching during development: a sync error
that keeps growing means the slew rate cannot keep up with hardware drift, and an RTT
that spikes means the main process is busy enough to poison the resync samples.

A cheap visual check is worth building alongside them — a marker that sweeps the screen
on a fixed period and crosses a tick on each exact second will expose audio/video drift
to the naked eye long before the numbers make it obvious.

See also: [[playback-clock]], [[api-reference]]
