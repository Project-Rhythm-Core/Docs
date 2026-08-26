---
title: Judgement
tags:
  - gameplay
  - timing
---

Judgement replicates **osu!mania's lazer / ScoreV2 model**. Charts imported from osu are
meant to play as they do in osu, so the windows and the hold-note rules are taken from
osu's own specification rather than invented.

## Which osu, and why it matters less than it looks

Three variants exist, and they agree more than the discussion around them suggests.

Every window below PERFECT is **identical in all three**. lazer stores its windows as
ranges — `(64, 49, 34)` for GREAT, evaluated at OD 0, 5 and 10 — and interpolating those
gives exactly the `64 − 3 × OD` that the wiki documents for stable. The same holds for
GOOD, OK, MEH and MISS.

Only PERFECT differs:

| Variant | PERFECT half-window |
| --- | --- |
| Stable ScoreV1 | `16`, flat, regardless of OD |
| Stable ScoreV2, and lazer by default | `22.4 − 0.6 × OD` for OD ≤ 5, `24.9 − 1.1 × OD` for OD ≥ 5 |
| lazer with the Classic mod | reverts to the ScoreV1 windows |

On a typical chart the gap is a millisecond. At OD 7.5 the two coincide exactly.

**The real divergence is hold notes**, and it is not subtle. ScoreV1 gives a hold a
single judgement derived from a *combined* error — head error plus release error — with a
different multiplier at each tier. ScoreV2 and lazer treat the head and the tail as two
ordinary notes that happen to belong together.

The second model is what this implements, and it is also why the runtime note structure
looks the way it does: a hold's head and tail are simply two things to judge in the same
lane, which the [[chart-format/index#Judgement walks lanes|per-lane cursor]] already
handles without a special case.

## Windows

Half-widths in milliseconds, so a window of 40 accepts ±40 around the note.

| Judgement | Half-window | Weight |
| --- | --- | --- |
| PERFECT | `22.4 → 19.4 → 13.9` across OD 0 → 5 → 10 | 305 |
| GREAT | `64 − 3 × OD` | 300 |
| GOOD | `97 − 3 × OD` | 200 |
| OK | `127 − 3 × OD` | 100 |
| MEH | `151 − 3 × OD` | 50 |
| MISS | `188 − 3 × OD` | 0 |

Two details that are easy to get wrong and that the implementation follows exactly:

- **Every window is floored and then given half a millisecond back.** osu computes
  `floor(window) + 0.5`, so a window quoted as 40 ms really accepts 40.5. Skip this and
  every boundary is off by up to a millisecond in the player's disfavour.
- **Accuracy weights PERFECT above GREAT.** Under ScoreV1 the two are equal, so an
  all-GREAT play reads as 100 % there and as 300/305 here.

Interpolation hinges on OD 5 rather than averaging the endpoints, so each half of the
range has its own slope.

## The two asymmetries

Neither of these is symmetric around the note, and both come from osu rather than from
first principles.

**A late MEH is impossible.** Once the OK window closes the note is gone. MEH and MISS
exist only on the early side, which means a note is written off at `+OK`, not at `+MISS`.

**Pressing before the MISS window does nothing at all.** It is not a miss and it does not
consume the note; the input passes straight through and the note stays where it is.
Between the MISS and MEH boundaries an early press *is* a miss — that band is real, it is
just early-only.

```
        MISS      MEH   OK  GOOD GREAT PERFECT GREAT GOOD  OK
   ──────┼─────────┼─────┼────┼────┼───┬───┼────┼────┼────┼──────────►
   ignored                              note                    gone
```

## Hold notes

A hold is judged twice.

- **The head** behaves exactly like a tap.
- **The tail** is judged on release, with every window multiplied by **1.5**. An error
  that would be a GOOD on a tap is still a GREAT on a release.

Two things cap the tail at MEH, however cleanly it is released: the head having been
missed, and the body having been let go early. Letting go before the tail is reachable is
a *hold break* — nothing is judged at that moment, and the tail is left to be written off
when its own window closes.

A hold whose head was missed can still be grabbed late. It cannot score above MEH, but the
tail is there to be released, which matches lazer.

Because a hold is not finished until its release is judged, its lane's cursor stays parked
on it for the whole duration. That is what stops the lane from advancing to the next note
while a key is still down.

## Input timing

A press is timed by **when the event happened**, not by when the handler ran.
`KeyboardEvent.timeStamp` shares its clock with `performance.now()`, which is the clock
the [[audio-engine/renderer-clock|renderer's audio clock]] anchors to, so a keypress
converts to a song position by subtraction:

```ts
const ageMs = performance.now() - event.timeStamp;
return clock.positionMs() - ageMs - inputOffsetMs;
```

Reading the clock inside the handler instead measures whenever the browser got round to
dispatching, and that jitter lands directly in the judgement.

> [!important] Input latency is a second, unrelated offset
> `audio.offsetMs` on a chart aligns chart time with audible audio. It says nothing about
> how long the input stack — USB polling, kernel, compositor — takes to deliver a
> keypress. That is a property of the player's machine, so it lives in settings, and it is
> not the number stored per chart.

## Verification

The windows and the hold rules are covered by unit tests written against the published
formulas rather than against the implementation.

End to end, an autoplay that presses at each note's exact time scores **324 consecutive
PERFECTs with nothing lower**, which is the check that matters for the timestamp chain: a
systematic offset anywhere between the keyboard event and the judgement would show up as
GREATs instead.

Sources: [osu!mania judgement system](https://osu.ppy.sh/wiki/en/Gameplay/Judgement/osu!mania),
[Overall difficulty](https://osu.ppy.sh/wiki/en/Beatmap/Overall_difficulty),
[Accuracy](https://osu.ppy.sh/wiki/en/Gameplay/Accuracy),
[`ManiaHitWindows.cs`](https://github.com/ppy/osu/blob/master/osu.Game.Rulesets.Mania/Scoring/ManiaHitWindows.cs),
[`DrawableHoldNoteTail.cs`](https://github.com/ppy/osu/blob/master/osu.Game.Rulesets.Mania/Objects/Drawables/DrawableHoldNoteTail.cs)
