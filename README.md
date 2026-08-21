# proto/dist-stack: stacking distortion stages

A prototype on top of [shorepine/amy#1116](https://github.com/shorepine/amy/pull/1116)
(per-osc distortion). The branch is two commits: the PR squashed into one base
commit (`2d6eb0c`) and the prototype itself (`a0d01f8`) - that commit's diff is
the whole feature and its message the full rationale. This file is the short tour.

## The problem it removes

In #1116, `GC`/`GF`/`GH` read as independent toggles on the wire but share one
type slot in the engine: `GC1` silently turns off an enabled crusher, and
printing state back to wire has to reconstruct which letter to emit.

## What it does

- `dist_config`'s type field becomes a stage bitmask; the `DIST_TYPE` delta
  splits into `DIST_CLIP_EN` / `DIST_FOLD_EN` / `DIST_CRUSH_EN`. Each command
  touches only its own stage and state-to-wire collapses to 1:1.
- The wire format is unchanged: existing patch strings mean the same thing,
  they just stop being lossy in the engine. Multi-stage messages like
  `GC1GH6,5` round-trip verbatim.
- Enabled stages run as their own passes over the block in a fixed
  clip -> fold -> crush order (shaping before lo-fi; the reverse order stays
  reachable by putting the crusher on a chain member and the clipper on its
  SILENT head). Every pass keeps the zero-overhead loop form, so cost is
  additive per enabled stage - roughly 70 cycles/sample/osc with all three
  on ESP32-S3.
- Only the crusher has state, so its enable toggle is the one that resets
  the sample-and-hold and DC blocker.

## Where to look

- `src/filters.c` - `dist_block` becomes the per-stage pass loop.
- `src/amy.h` / `src/amy.c` - the stage bitmask and the three enable deltas.
- `src/parse.c` / `src/patches.c` - parse and the 1:1 printer.

## Deliberately unchanged (the open design questions)

- Drive and mix stay shared across the chain; per-stage drive is where a
  per-stage coef vector would live
  (see [proto/dist-coef-rail-v2](https://github.com/rt-rtos/amy/tree/proto/dist-coef-rail-v2) for the
  shared-rail answer).
- Each pass crossfades against its own input by the shared mix; a single
  wet/dry wrap around the whole chain is the alternative, one scratch buffer
  away.

## Verified

17 parse/print round-trips: `GC1GH6,5` / `GC0GF1` round-trip verbatim, `GH0`
prints as `GH0` instead of the old canonical `GC0`, single-stage output is
byte-identical to #1116 (RMS pin 1174.8 -> 1528.8), clip+crush stacked
renders distinct from clip alone, and toggling the crusher back off restores
clip-alone output byte-exactly.
