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

## Demo

**10 - bus scope: one signal distorted BY another**
A quiet sustained
mid sine plus a loud low sine pulsing one second on, one off; identical clip
settings in both arms. Per-osc - even applied to both oscs - leaves the mid
tone static. On the bus the low tone drags the sum into the knee, so the mid
tone ducks and buzzes exactly while the low tone sounds

https://github.com/user-attachments/assets/f72bedee-0ae1-4f2a-9e30-ec4703fdf023



https://github.com/user-attachments/assets/addafedd-4c0c-442d-a159-dfb6d682b792



https://github.com/user-attachments/assets/e03a6299-c96f-449a-b69e-087ab46da5ce

**12 - bus drive ramp on a held triad.** The ET triad from demo 6, sustained,
with bus CLIP drive swept 1 to 16: the intermodulation fan blooms out of
three pure tones in one gesture



https://github.com/user-attachments/assets/d4083b94-71e6-4cd3-8328-5c997eabc2cd



https://github.com/user-attachments/assets/def46ab4-a1d2-4c41-9777-69af6e216ca5



**13 - chain position: crush before echo vs after.** 13a is the real chain -
the bus crusher feeds the bus echo, so every repeat is a scaled copy of the
crushed stab and the tail decays smoothly. 13b applies the same 5-bit
quantizer AFTER the echo (simulated: numpy quantization of the echoed dry
render, labeled as such) - the decaying repeats fall through the quantization
steps and gate out, exactly like demo 5's release tail. Why the stage sits
first in the bus FX chain.



https://github.com/user-attachments/assets/a492003d-9ca9-4a77-9787-d9133000d7b0



https://github.com/user-attachments/assets/2699e9e3-1de7-41ac-ac9e-2c225dcd791a



https://github.com/user-attachments/assets/c3321983-ed44-47ca-846f-0f18844a49fc



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
