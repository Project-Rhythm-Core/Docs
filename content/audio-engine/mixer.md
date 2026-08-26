---
title: Mixer
tags:
  - audio
  - rust
---

Most osu!mania charts in circulation make no sound of their own — the music is one file
and the notes are timing data. A large minority do the opposite: `AudioFilename: virtual`,
no track at all, and the entire soundtrack is samples the player triggers by hitting
notes. Of the reference set, **193 of 204 charts** are that second kind.

Both go through the same mixer, and everything lands in **one stream**. That is not
tidiness. A hit sound routed through a separate output would carry its own latency, and
the game would feel wrong however accurate its judgement was.

## Two kinds of sound

| | Scheduled | Triggered |
| --- | --- | --- |
| Where it comes from | The chart's BGM events | The player hitting a note |
| When it is known | At load | Whenever it happens |
| Placed | To the exact frame | At the head of the next buffer |

**Scheduled** sounds are fired by the audio thread itself. It holds the list, ordered, and
walks a cursor through it, so a sound authored for a given frame starts on that frame
rather than wherever the buffer boundary happened to fall.

**Triggered** sounds cannot be placed better than the next buffer: the press has already
happened by the time anything hears about it. They cross into the audio thread through a
lock-free queue, because the callback must never block, allocate, or wait on another
thread. A full queue drops the sound rather than stalling — a missing sound is bad, a
stalled audio thread is much worse.

## Voices

64 of them, fixed, allocated once. Piano charts sustain heavily, so running out is
plausible; when it happens the **oldest** voice is stolen rather than the newest being
refused. Cutting a note that has been ringing for a while is far less noticeable than
silencing the one the player just played.

A seek cuts every voice. Leaving the previous position still sounding would be worse than
the gap.

`PlaybackStats.activeVoices` reports how many are sounding, and `droppedSamples` counts
what the queue could not take. The second should stay at zero.

## Things that were silently wrong

Three failures in this area produced no error at all, only silence. All three are worth
knowing about because nothing about them announces itself.

> [!caution] Containers and codecs are separate features
> In Symphonia, `ogg` and `wav` provide only the *readers*. Without `vorbis` and `pcm`
> every `.ogg` and `.wav` in a sample bank fails to decode — which is most of them. The
> chart loads, plays, and makes no sound.

> [!caution] Charts disagree with their own files about case
> osu runs on case-insensitive filesystems, so a chart naming `normal-hitnormal.wav` while
> shipping `Normal-Hitnormal.wav` is ordinary rather than broken. On Linux that reference
> resolves to nothing. Sample resolution indexes the chart folder by lowercase name and
> matches against that, which handles the extension mismatch in the same pass.

> [!caution] An empty range still panics if it starts past the end
> A keysounded chart has a one-frame placeholder where its music would be, while its
> cursor runs the length of the whole chart — so the cursor is past the end almost
> immediately. `samples[start..start]` panics when `start` exceeds the length, empty range
> or not, and a panic on the audio thread kills it outright: playback continues, position
> keeps advancing, and nothing ever sounds again.

A sample that cannot be decoded is loaded as silence rather than failing the chart, since
charts routinely name sounds they do not ship. `AudioInfo.silentSamples` reports how many,
so that stays visible instead of being swallowed.

> [!note] Missing skin sounds have no fallback yet
> A chart naming a sound it does not ship expects the player's skin to supply it. There is
> no built-in skin, so those notes are silent. That is the remaining gap here.

See also: [[native-module]], [[playback-clock]], [[chart-format/index#Some games have a song, others have samples|the chart format's two audio models]]
