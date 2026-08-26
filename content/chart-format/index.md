---
title: Chart Format
tags:
  - charts
  - format
---

The game reads one chart format. Everything else — osu!mania, BMS, o2jam, pop'n — is
converted into it **at import time**, never at play time. The runtime never learns what a
`.osu` file is.

## Why convert at all

Supporting several source formats by teaching the game to read each one means every
gameplay feature has to handle every format's quirks forever. Converting once puts all
that complexity in an importer, which runs offline, can be tested against fixtures, and
can be rewritten without touching gameplay.

The cost is that the format must span the *union* of what those games need, not the
intersection. A format designed around osu!mania alone is a list of `{time, column}` — and
retrofitting keysounds into that is not a migration, it is a rewrite.

## The two axes that actually matter

Everything awkward about VSRG formats comes from two distinctions the source formats blur.

### Musical time is not scroll position

Where a note is *judged* and where it is *drawn* follow different timelines. A note at
10 000 ms is judged at 10 000 ms regardless of scroll velocity, but its position on screen
depends on every velocity change between now and then.

osu packs both into `[TimingPoints]`, distinguished by an `uninherited` flag. BMS uses
`#BPM` for one and `#STOP` plus scroll extensions for the other. This format splits them:

- **`timing.tempo`** — the musical grid. BPM and meter.
- **`timing.scroll`** — a velocity multiplier applied on top of it.
- **`timing.stops`** — scroll freezes that do not stop musical time.

Scroll *positions* are deliberately **not** stored. They are derived at load in one pass,
which keeps the file canonical and makes a "constant velocity" mod a matter of ignoring
the velocity data rather than shipping a second chart.

> [!important] Tempo drives scrolling too — it is not just musical metadata
> The obvious reading of the split above is that `timing.scroll` alone determines how
> fast notes move. That is wrong for osu!mania, where scroll velocity follows **BPM as
> well as the multiplier**, and mappers exploit it.
>
> The evidence is unambiguous. One reference chart ships an SV version and a "no SV"
> version with **identical note times**. The difference between them is almost entirely
> in the tempo timeline: 1528 tempo points spanning 234 distinct BPMs — from 140 up to
> **6 942 069** — against just 8 distinct scroll multipliers. The visual effects are
> authored as tempo changes, not as velocity changes.
>
> A derivation that reads only `timing.scroll` renders that chart as if it had no effects
> at all. Keeping the two timelines separate is still right, because they mean different
> things and other formats drive them independently — but the derivation has to consume
> both.

### Some games have a song, others have samples

| Model | Games | How audio works |
| --- | --- | --- |
| Background track | osu!mania, usually | One long file. Notes are timing data only. |
| Keysounded | BMS, o2jam, pop'n (PMS), **and osu!mania** | Hundreds of samples. **Pressing the key is what plays the sound.** |

osu!mania sits on both sides of that line. A chart with `AudioFilename: virtual` has no
background track at all: every sound comes from the notes and from scheduled samples. The
reference pack is 193 such charts, so the mixer is a prerequisite well before BMS enters
the picture.

In a keysounded chart there is no "the song": the music is the samples your hits trigger
plus the autoplay BGM channels. Miss a note and that sound does not happen.

The format carries both, and a chart may use either or both:

- **`audio`** — an optional background track, streamed from memory.
- **`samples`** — a sample bank.
- **`bgmEvents`** — samples that fire on schedule with no note attached.

The bank is named `samples` rather than `keysounds` because it covers both cases. osu
hitsounds and BMS keysounds resolve identically: a note names sounds that fire when it is
hit. What differs is only whether the result is decoration over a track or the music
itself.

> [!important] This is a requirement on the audio engine, not just the format
> A keysounded chart needs a polyphonic mixer with a sample bank resident and low-latency
> triggering from input. The [[audio-engine/index|current engine]] plays one track through
> one voice, which is enough for osu!mania and cannot play BMS at all. Input-to-sound
> latency also stops being merely a scoring matter and becomes directly audible.

## Schema

```json
{
  "formatVersion": 1,
  "id": "osu:5633412",

  "metadata": {
    "title": "RoquiRa : Between Life and Death",
    "titleUnicode": "RoquiRa : Between Life and Death",
    "artist": "Raphiiel",
    "artistUnicode": "Raphiiel",
    "charter": "Rurvker",
    "difficultyName": "Finale : The End of Destination",
    "source": "",
    "tags": ["marathon", "speedcore"]
  },

  "origin": {
    "format": "osu",
    "formatVersion": 14,
    "ids": { "beatmapId": 5633412, "beatmapSetId": 2543786 }
  },

  "columns": [
    { "role": "note" }, { "role": "note" }, { "role": "note" }, { "role": "note" },
    { "role": "note" }, { "role": "note" }, { "role": "note" }, { "role": "note" },
    { "role": "note" }, { "role": "note" }
  ],

  "audio": {
    "file": "audio.mp3",
    "offsetMs": 0,
    "previewMs": 439318,
    "leadInMs": 0
  },

  "samples": [ { "file": "soft-hitnormal.wav" } ],
  "bgmEvents": [ { "timeMs": 2550, "sample": 0, "volume": 91 } ],

  "timing": {
    "tempo":  [ { "timeMs": 1292, "bpm": 183.0, "meter": 4 } ],
    "scroll": [ { "timeMs": 1292, "multiplier": 1.0 } ],
    "stops":  []
  },

  "notes": [
    { "timeMs": 964,  "column": 6, "samples": [0], "volume": 40 },
    { "timeMs": 1292, "column": 7, "endMs": 1619, "samples": [0], "volume": 40 }
  ],

  "effects": [ { "startMs": 95718, "endMs": 116701, "kind": "kiai" } ],
  "breaks":  [ { "startMs": 371326, "endMs": 374844 } ]
}
```

### Top level

| Field | Type | Notes |
| --- | --- | --- |
| `formatVersion` | integer | Present from version 1. Bump on any breaking change. |
| `id` | string | Stable identity, namespaced by source (`osu:5633412`, `bms:<md5>`). |
| `origin` | object | Provenance. Keep it: re-importing and debugging both need it. |
| `columns` | array | One entry per lane, in play order. Its length *is* the key count. |
| `audio` | object \| null | Background track, when there is one. |
| `samples` | array | Sample bank: `{ file }` entries, referenced by index. |
| `bgmEvents` | array | `{ timeMs, sample, volume }`. Samples that play with no note to hit. |
| `timing` | object | `tempo`, `scroll`, `stops`. |
| `notes` | array | The chart. |
| `effects`, `breaks` | array | Presentation only; never affects judgement. |

### `columns[]`

| Field | Type | Notes |
| --- | --- | --- |
| `role` | `"note"` \| `"scratch"` | A turntable is not a narrower note lane — it is judged and rendered differently. BMS and PMS need this; osu!mania sets everything to `note`. |

Key count is `columns.length`. There is no `keys` field, because a number invites
hardcoding 4 and the first real chart tested against this format is 10K.

### `notes[]`

Kept small on purpose: a marathon chart is ~12 000 entries, and optional fields are
omitted rather than nulled.

| Field | Type | Notes |
| --- | --- | --- |
| `timeMs` | number | Absolute milliseconds from the start of the audio. |
| `column` | integer | Index into `columns`. |
| `endMs` | number? | Present ⇒ hold note. |
| `kind` | `"mine"`? | Omitted for ordinary notes. BMS landmines set it. |
| `samples` | integer[]? | Indices into `samples`, played together. Omitted when empty. |
| `volume` | number? | 0 to 100. Omitted when full. |

`samples` is a list, not a single value, because osu hitsounds layer: one note can fire a
normal sound plus a whistle and a clap at the same instant.

Volume lives on the note rather than on a timeline. Source formats allow it per hit
object — the reference pack varies it note by note across the whole range — so a timeline
alone cannot represent those charts. Resolving it at import also means the runtime plays
a sound without looking anything up.

### `timing`

| Array | Field | Notes |
| --- | --- | --- |
| `tempo[]` | `timeMs`, `bpm`, `meter` | The musical grid. |
| `scroll[]` | `timeMs`, `multiplier` | Visual velocity. `1.0` is the chart's base speed. |
| `stops[]` | `timeMs`, `durationMs` | Scroll freezes. Musical time continues. |

## Invariants

The importer guarantees these; the runtime is allowed to rely on them without checking.

1. `notes`, and every timeline array, are sorted ascending by time.
2. Times are rounded to microseconds and rates to six decimals, so a value that is
   arithmetically 187 is written as `187`, not `187.00000000000003`. Source formats store
   milliseconds per beat, so every tempo is a division and arrives with that noise.
3. Every `column` is in `[0, columns.length)`.
4. For a hold, `endMs > timeMs`.
5. No two notes share the same `timeMs` **and** `column`.
6. Every `notes[].samples` and `bgmEvents[].sample` indexes a valid `samples` entry.
7. Each `timing` array has an entry at or before the first note, so lookups never fall off
   the front.

An importer that cannot satisfy these should fail loudly rather than emit a chart the
runtime will misread.

## Mapping osu!mania

Verified against a 10K marathon: 11 733 notes, 9 BPM changes, 43 velocity points.

| Source | Rule |
| --- | --- |
| Mode | `General.Mode` must be `3`; anything else is not a VSRG chart. |
| Key count | `Difficulty.CircleSize`. In mania this is the lane count, not a size. |
| Column | `floor(x * columnCount / 512)`. The `y` coordinate is meaningless in mania. |
| Tap | `type & 1` |
| Hold | `type & 128`; `endMs` is the first field of the trailing sample group, before the colon. |
| Tempo | Timing point with `uninherited == 1`; `bpm = 60000 / beatLength`. |
| Velocity | Timing point with `uninherited == 0`; `multiplier = -100 / beatLength`. |
| Kiai | `effects & 1`, spanning until the next point that clears it. |
| Breaks | `[Events]` lines of type `2`. |
| Note sounds | See below. |

### osu hitsounds

Most mania charts do not name sample files per note. They rely on **sample sets**: a
three-level inheritance chain resolved into a filename.

1. A hit object's own `normalSet` / `additionSet` / `index`, where `0` means inherit.
2. The timing point in force, which carries a set, an index and a volume.
3. `[General] SampleSet` as the final fallback.

The resulting name is `{set}-hit{sound}{index}.wav`, with index `1` written as nothing at
all. The reference chart declares Soft at index 1 with every hit object inheriting, so all
11 733 notes resolve to a single `soft-hitnormal.wav`.

The `hitSound` bitfield then layers additions on top: whistle (2), finish (4), clap (8).
Bit 0 is implied — a value of `0` still plays the normal sound. A note with `10` fires
three samples at once, which is why `notes[].samples` is a list.

A custom filename in the last slot of the sample group replaces the set-derived sound
entirely. Keysounded charts use nothing else: all 247 568 hit objects in the reference
pack name their own file.

### Scheduled samples

Storyboard sample events — `Sample,time,layer,"file",volume` in `[Events]` — are how a
keysounded osu chart carries everything the player does not hit. They map straight onto
`bgmEvents`.

They are easy to overlook and expensive to miss: the reference pack has **26 428 of them
across 152 charts**. An importer that skips them produces charts that are nearly silent
between notes.

### Filenames do not have to match what is on disk

A chart may name `A7S.wav` while shipping `A7S.ogg`. osu resolves this by trying other
audio extensions, and the importer does the same, storing the name that will actually
open. In the reference pack **170 of 334 references** need this rewrite.

> [!note] Some references resolve to nothing at all
> Charts name sounds they do not ship — either expecting the player's skin to supply them,
> or as leftovers. Fourteen references across the reference pack resolve to nothing,
> including entries as obviously broken as `null_s.wav` and a literal `.wav`. The importer
> keeps the name as written; the loader has to tolerate a miss rather than fail the song.

Two rules are less obvious than they look, and both were found by running the importer
against a real chart rather than by reading the specification:

- **An uninherited point also resets velocity to 1.0x.** It is a tempo point *and* a
  scroll point, so it emits an entry in both arrays. Missing this leaves an old velocity
  applied across a BPM change.
- **The sign of `beatLength` outranks the `uninherited` flag.** A negative value cannot be
  a duration per beat, so a point claiming to be uninherited while carrying one is
  contradicting itself. SV charts do write these — one reference chart has
  `164135,-100000000000000,4,1,0,20,1,0`, which is a near-total scroll freeze wearing a
  tempo point's flag. Trusting the flag over the sign fails the import; trusting the sign
  produces what the mapper meant. Old files omit the flag entirely, where the same rule is
  the only thing to go on.
- **A nonsensical velocity is skipped, not fatal.** Velocity is presentation: if the
  arithmetic yields something unusable, the previous multiplier stays in force and the
  chart still loads. A nonsensical *tempo* does fail the import, because timing is
  load-bearing.
- **Charts open before their first timing point.** The reference chart's first note is at
  964 ms while timing begins at 1292 ms. osu treats that region as governed by the first
  point, so the importer prepends a copy at the earliest note. Moving the original
  instead would shift the beat grid for the whole chart.

Parsing notes: files are CRLF, may carry a UTF-8 BOM, and sections can be missing
entirely. A section header can also have its first record stuck to it on the same line —
`[TimingPoints]0,500,4,1,0,100,1,0` — which the importer recovers from, since the `]`
closes the name unambiguously. One chart in the reference pack does this, and refusing it
would fail an import for a formatting quirk that costs nothing to absorb. `SliderMultiplier` is an osu!standard concept that does not drive mania scroll —
that comes from the player's speed setting modulated by `scroll[].multiplier` — so it is
kept in `origin.values` rather than being given a field of its own.



## Deriving scroll positions

`src/lib/chart/scroll-timeline.ts` turns the two timelines into the positions the renderer
draws at. Positions are in **reference milliseconds**: a note 1000 units away is one
second away at the chart's base tempo and unmodified velocity, which the renderer scales
to pixels with the player's speed setting.

Velocity is `(bpm / referenceBpm) × multiplier`, and the reference is the chart's first
tempo point, so its opening section scrolls at exactly the speed the player chose. The two
timelines merge into stretches of constant velocity, and position is the exact
piecewise-linear integral over them — never a per-frame accumulation, which would drift.

`constantVelocity` ignores everything the chart authored, which is the whole "no SV" mod.
Because positions are derived rather than stored, that costs nothing.

### What the effects actually are

Extreme values are not corruption. They are how mappers build effects that osu's own
editor cannot express, generated with external tools:

- **Notes that are never seen.** Velocity high enough that a note crosses the playfield in
  less than a frame.
- **Notes that sit still.** Velocity at or near zero, so a note holds position for seconds
  and still has to be hit on time.

Running the derivation over the reference SV chart reproduces both, and its own "no SV"
sibling — same notes, same times — behaves as a clean control:

| | notes on screen < 1 frame | notes on screen > 60 s | ordinary |
| --- | --- | --- | --- |
| SV version | 431 (29.6 %) | 273 (18.8 %) | 750 |
| no-SV version | 0 | 0 | 1454 |

> [!important] Judgement must not depend on rendering
> Roughly a third of that chart is unseeable and a fifth is motionless, yet every note is
> judged normally. Anything that culls, schedules or judges by screen position breaks
> those charts outright. Judgement reads `note.timeMs`; only drawing reads positions.
>
> The same follows for ordering. With velocity at zero, many notes share a position, so
> scroll position is not an ordering — time is.

> [!caution] Do not sanitise authored values
> Clamping velocity to a "sane" range, or rejecting an implausible BPM, deletes the chart's
> content. The importer stores what the mapper wrote; the derivation only has to stay
> finite.

## The runtime shape is not the file shape

The file stores notes flat and sorted by time. That is right for something a parser writes
and a human reads, and wrong for playing: gameplay has two access patterns that pull in
different directions, so `PlayableChart` builds both views at load.

Neither view copies a note. Both hold indices into `chart.notes`, so note state has one
home.

### Drawing walks a window

Velocity is never negative, so scroll position never decreases with time, so notes sorted
by time are also sorted by position. That is what makes the visible set a **contiguous
range**, which one cursor can track as it advances.

Worth remembering, because it is an assumption rather than a law: a format that allowed
reverse scrolling would break it, and BMS extensions do exist that allow exactly that.

A hold stays visible while any part of it is on screen, so the near edge of the window is
tested against tails rather than heads.

### Judgement walks lanes

A press in one lane can only hit a note in that lane, and only the earliest unjudged one.
So each column carries its own cursor into its own list of note indices, and resolving a
press is a lookup rather than a search.

Two details matter more than they look:

- **It is *unjudged*, not *unhit*.** A note that expires is judged — as a miss — without
  ever being hit, and the cursor has to move past it. Retiring expired notes each frame is
  what keeps the cursor honest when the player does nothing.
- **Retirement runs on time, never on position.** A chart can hide notes entirely or
  freeze them in place. Both still have to be judged.

> [!important] Judgement must not consult the renderer
> On the reference SV chart, roughly a third of the notes cross the screen in under a
> frame and a fifth never move at all, yet every one is judged normally. Playing it back
> confirms the separation holds: through a section where velocity sits at 7×10⁻⁶ and notes
> pile up motionless on screen, the miss count keeps climbing at the same rate as it did
> while scrolling normally.
>
> The same follows for ordering. With velocity at zero, many notes share a scroll
> position, so position is not an ordering — time is.

## Importers

| Source | Entry point | State |
| --- | --- | --- |
| osu!mania | `convertOsuChart(sourcePath, outputPath)` | Implemented, background-track and keysounded |
| BMS / PMS | — | Blocked on the mixer |
| o2jam | — | Not started |

`convertOsuChart` parses on a worker thread, writes the JSON, and resolves with a summary
rather than the chart itself: marshalling tens of thousands of notes across the N-API
boundary would cost far more than writing the file. It refuses anything that is not mode
3, and anything that violates the invariants above.

Sample references are resolved against the chart's own folder, so `import` needs the file
path rather than its contents.

Measured against the two reference sets:

| Set | What it exercises | Result |
| --- | --- | --- |
| 10K marathon, background track | Volume and scale | 11 733 notes in ~10 ms, ~1.2 MB of JSON |
| 193 keysounded 7K charts | No background track, BGM events, extension rewrites | 247 568 notes, 26 428 BGM events, ~220 ms |
| 4K SV showcase | Extreme tempo and velocity | BPM 0.001 – 6 942 069, velocity 0 – 10x |
| 7K full long notes | Holds at 100 % | 2 071 notes, every one a hold |
| 7K dense | Conversion under load | 10 600 notes, 5 000 holds |
| 10K hybrid | Background track *and* keysounds | Both sample mechanisms in one chart |

Every chart above round-trips with **zero note mismatches** against an independent parse
of the source.

The hybrid set is the one that exercises sample resolution end to end: a background track,
two different sample sets in the same chart, 817 notes on set-derived sounds, 265 naming
their own file, 169 firing more than one sample at once, and seven distinct volumes.

## What the other formats will add

| Format | New demands |
| --- | --- |
| BMS / PMS | Keysounds and BGM channels; measure-based time; `#STOP`; scratch lane; landmines; `#RANDOM` branches resolved at import with a seed; 5/7/9/10/14 keys |
| o2jam | Binary `.ojn` + `.ojm` sample bank, some variants encrypted; three difficulties per file |
| pop'n | Reaches the format through PMS, so BMS support covers it |

Nothing above needs a schema change — which is the point of designing for the union now.

## The offset problem

A chart's timing was authored against *the importing game's* audio pipeline. That pipeline
is not necessarily this one.

The engine trims encoder delay — 1105 samples, **25 ms**, on a typical LAME MP3 (see
[[audio-engine/native-module#Encoder delay]]). If the game a chart came from did not trim
it the same way, every note in that chart lands systematically early or late by roughly
that much: comfortably inside a "Perfect" window.

This is what `audio.offsetMs` is for. Two rules follow:

- **Measure it, do not assume it.** Verify one chart per source format against real
  playback before trusting an importer's output.
- **Store it per chart, not per game.** It depends on how each individual audio file was
  encoded, so a single global constant will be wrong for some files.

> [!warning] Judgement needs a second, unrelated offset
> `audio.offsetMs` aligns chart time with audible audio. It says nothing about how long
> the input stack takes to deliver a keypress. That is a player and hardware property, not
> a chart property, so it belongs in settings — never in the chart file.

## Practical notes

- **Song select should not parse charts.** Reading 12 000 notes to draw a list entry does
  not scale. Emit a separate metadata-only index at import.
- **JSON is the right trade here.** A marathon chart is a few hundred kilobytes and parses
  in milliseconds, and being diffable and hand-editable is worth more than the bytes.
  Revisit only if profiling says so, and then only for the note array.
- **Charts reference audio, they do not contain it.** Keep converted charts and their
  media outside the source tree.
