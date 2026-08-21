# proto/dist-bus-stage: per-bus distortion

A prototype on top of [shorepine/amy#1116](https://github.com/shorepine/amy/pull/1116)
(per-osc distortion). The branch is two commits: the PR squashed into one base
commit (`2d6eb0c`) and the prototype itself (`b96d4a7`) - that commit's diff is
the whole feature and its message the full rationale. This file is the short tour.

## What it does

Runs `dist_block` over each bus's oscillator sum, first in the bus FX chain
(before EQ / chorus / echo / reverb), so the delays and reverb take tails of the
shaped signal rather than the other way round. The bus level scales with
polyphony and that is kept deliberately: responding to bus level is what a
mixbus saturator is for. Sparse sources duck under a dense pad, chords saturate
harder than single notes.

## Wire

`J` mirrors `G`'s sub-command grammar at bus scope: `JC`/`JF`/`JH` pick the
stage, `JD` drive, `JM` mix. Bus-directed deltas ride the established `y`
prefix, naming their bus the way `VOLUME`/`EQ`/`CHORUS`/`ECHO`/`REVERB` do.
The letter choice extrapolates the `G` sub-command pattern to a second scope -
flagged as a convention guess to confirm in review. Python mirrors it as
`bus_dist_*` kwargs.

## Where to look

- `src/filters.c` - the bus entry point over the stereo bus sum; channels get
  their own `dist_state`, per `dist_block`'s contract.
- `src/amy.c` - the call site in the bus FX chain; params clamp once at delta
  apply so the render path stays check-free.
- `src/parse.c` / `src/patches.c` - `J` parse and state-to-wire readback.

## Numerics

The pre-gain moves from `MUL6A_SS` to `SMULR6`, following the codebase's move
to the 64-bit multiply: a bus sum runs several times full scale, so
`drive * x` can pass `MUL6A_SS`'s [-64, 64) product range and wrap sign.
`SMULR6` is exact where hardware allows (ESP32-S3, desktop); the 32x32
fallback keeps [-128, 128), and the stage pre-clamps its input to 8x full
scale as the matching wrap guard - the output mixdown clips a bus at 10x
anyway, and the shaper saturates everything above 1/drive, so only drive < 1
on a hotter-than-8x bus could notice.

## Verified

On the host module: enable-then-disable renders bit-identical to
never-enabled; clip at drive 8 lifts a saw voice's RMS 0.030 -> 0.054; a
bus-1-directed config leaves bus 0 byte-identical and shapes bus 1; crush
engages through the same path; an 8-voice near-full-scale bus at drive 16
stays bounded.
