---
title: Latency and Calibration
tags:
  - audio
  - timing
  - latency
---

Four distinct delays sit between "the user pressed play" and "the player hears the beat".
The engine attacks each one differently — three are engineered away, the fourth cannot
be, and gets a manual knob.

| Delay | Magnitude | Handled by |
| --- | --- | --- |
| Decode the file | hundreds of ms | Moved off the critical path — worker thread at load |
| Open the audio device | hundreds of ms | Moved off the critical path — stream starts at load |
| Device buffer period | ~5 ms | Minimised — 256-frame buffers |
| Speaker / DSP / Bluetooth | 0–200+ ms, unknowable | Manual offset |

## Warm-up: paying the cost early

`AudioEngine::prepare` opens the device and calls `stream.play()` immediately, before
the user has done anything. The callback runs from that moment, filling every buffer
with zeros because `armed` is still `false`.

This turns the expensive part of playback into load-time work:

```
Naive:  [press play] ─── open device (250 ms) ─── first sound
Engine: [load] ─ open device (250 ms) ─ silence… [press play] ─ 5 ms ─ first sound
```

In practice this means loading a chart's audio during the song-select or countdown
screen, not when the chart begins.

`start()` then only flips an atomic, so audio begins on the next callback — one buffer
period.

### Knowing when it is safe

The device is not instantly useful after `stream.play()`. ALSA takes time to actually
spin up, and during that window callbacks arrive without the device queueing ahead. The
engine detects the transition inside the callback:

```rust
if estimated_lead_nanos > 0 {
    clock.mark_device_warm();
}
```

A positive estimated lead means the frames already handed over will still be draining
when the next callback arrives — the device is genuinely running ahead. That is exposed
as `isReady()` / `PlaybackStats.ready`.

> [!note] Warm is not the same as settled
> The flag marks the moment the device *starts* queueing ahead; it keeps filling for a
> while afterwards. That distinction matters for the hardware rate estimate, which waits
> a further two seconds before taking its reference — see [[playback-clock#Hardware rate]].

The renderer waits for it explicitly before enabling the play button:

```ts
await clock.load(chartAudioPath);
if (!(await clock.waitUntilReady())) {
  // The device never came up; surface it rather than starting a chart that will drift.
  return;
}
```

> [!important] Calling `play()` cold
> Nothing prevents `play()` before `isReady()` — it just sounds late, by up to ~200 ms,
> because ALSA is still bringing the stream up. `waitUntilReady()` polls every 20 ms
> with a 5 s default budget and returns `false` rather than throwing on timeout.

## Buffer period

`PREFERRED_BUFFER_FRAMES = 256` is the deliberate floor on output latency:

| Sample rate | 256 frames |
| --- | --- |
| 44 100 Hz | 5.80 ms |
| 48 000 Hz | 5.33 ms |
| 96 000 Hz | 2.67 ms |

The constant is `PREFERRED_BUFFER_FRAMES` in `rust-core/src/audio/device.rs`.

The system default would typically be 1024–8192 frames (20–170 ms), which alone would
exceed a rhythm game's entire timing window. The trade is CPU: smaller buffers mean more
callbacks per second and less headroom before an xrun.

`AudioInfo.bufferFrames` reports what the device actually granted after clamping into
its supported range — `0` means the backend chose its own and `bufferMs` will be `0` too.

## Output latency

`PlaybackStats.outputLatencyMs` is the lead latched on the first armed callback: how far
in the future that buffer's first frame would be audible. It varies enormously with the
device path — on one Linux machine the same hardware reports ~14 ms through one backend
and ~3.5 ms through another.

It is already accounted for in `positionMs()` — the clock is anchored to the *audible*
instant, not the submission instant — so it is a diagnostic, not something to compensate
for again. Doing so would double-count.

## The manual offset

No API knows the real latency of the physical output chain. Bluetooth headphones add
100–200 ms. An AV receiver's DSP adds tens. A TV's picture processing shifts the video
instead. None of it is reported anywhere.

That residual is `setOffsetMs()`:

```rust
pub fn set_calibration_offset_ms(&self, offset_ms: f64) {
    self.calibration_offset_nanos
        .store((offset_ms * 1_000_000.0) as i64, Ordering::Relaxed);
}
```

It is subtracted inside `position_ms()`, so it shifts the reported position rather than
the audio:

```
elapsed = now − origin − offset
```

**Sign convention:** *positive* offset means the audio is heard **later** than the clock
claims, so the reported position must lag. If notes appear to arrive before the sound,
increase the offset.

`setOffsetMs()` can be called at any time, including mid-playback, so a calibration
screen can nudge the value live while the player listens.

> [!tip] What the application still owes the player
> The engine provides the knob, nothing more. A shipping build needs a calibration flow
> (a metronome the player taps against is the usual approach) and **persistence per
> output device** — the correct offset for wired headphones is not the correct offset
> for the same machine's Bluetooth ones, and the engine keeps no state across loads.

Note the asymmetry: `setOffsetMs` is applied natively, but the renderer's `AudioClock`
picks it up implicitly, because its handshake reads the already-offset `position()`. A
change mid-playback is absorbed by the next resync — instantly if it exceeds the 80 ms
snap threshold, gradually otherwise.

## Keeping the renderer honest

Low audio latency is wasted if Chromium throttles the renderer. `main.cjs` disables
every relevant throttle:

```js
app.commandLine.appendSwitch('disable-background-timer-throttling');
app.commandLine.appendSwitch('disable-renderer-backgrounding');
app.commandLine.appendSwitch('disable-backgrounding-occluded-windows');
```

plus `backgroundThrottling: false` in `webPreferences`. Without these, alt-tabbing away
throttles `requestAnimationFrame` and timers, and video desynchronises from audio the
moment the window loses focus.

`will-quit` also releases the device explicitly — otherwise it can be left held after the
process exits.

> [!note] `sandbox: false` on the window
> The preload script needs `require` for relative modules, so it can share the channel
> list with the main process; the renderer sandbox does not provide it. Disabling the
> sandbox applies to the preload only — the page itself still has no Node access, and
> `contextIsolation` keeps preload internals out of its reach. This is safe while the
> window only ever loads local content. The alternative is duplicating the channel names
> into the preload, or bundling it.

See also: [[playback-clock]], [[renderer-clock]]
