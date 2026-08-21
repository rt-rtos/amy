# proto/dist-coef-rail-v2: drive and mix as control coefficients

A prototype on top of [shorepine/amy#1116](https://github.com/shorepine/amy/pull/1116)
(per-osc distortion) and [proto/dist-stack](https://github.com/rt-rtos/amy/tree/proto/dist-stack). The
branch is three commits: the PR squashed into one base commit (`2d6eb0c`), the
stacking prototype (`a0d01f8`, see that branch's README), and this feature
(`2c3ae61`) - the tip commit's diff is the whole feature and its message the
full rationale. This file is the short tour.

## Why

Drive is a timbre control, so it wants the rail every other timbre control in
AMY has. Velocity into drive, EG1 into drive and a mod source into drive are
what make a waveshaper part of a voice rather than an insert effect, and none
of them are reachable while drive is a scalar fixed at delta apply.

## What it does

- `GD` and `GM` grow from scalars into control-coef lists. The wire does not
  change shape: `parse_coef_message` on a single value sets just the constant
  term, so `GD8` means what it meant. A list adds modulation: `GD1,,2` is
  unity drive plus two octaves at full velocity.
- Drive rides a log2 rail, exactly as freq and filter freq do: the CONST coef
  stays in linear drive (1 is unity) and the modulation coefs are octaves of
  it. An octave is also the natural unit for FOLD, where it buys one more
  fold. The rail spans 2^-4..2^4; mix, not drive, is how the stage turns
  down. Mix follows duty instead: linear combine, clamped 0..1.
- Both combine in `hold_and_modify` into `msynth`, gated on `dist_stages`, so
  an osc with no stage enabled pays one compare as before. `dist_block`
  already hoists drive and mix above its sample loops, so the loops are
  untouched and stay zero-overhead; the config it receives is still checked
  once per block, never per sample.
- One shared vector serves the whole stage chain - a deliberate choice
  against per-stage drive vectors; per-stage character comes from which
  stages are enabled, and per-stage trims could be added later without
  disturbing this shape.
- `DIST_LOGDRIVE` and `DIST_MIX` claim ten param ids each out of the block
  `VOLUME` freed, leaving 97..98.

## Demo

**16a - drive rides velocity.** Drive = constant 1 plus 3 octaves of
velocity (`GD1,,3` on the wire) into CLIP: soft hits play a near-pure tone,
hard hits a square bark - the waveshaper responding to touch like part of
the voice, not an insert effect.

https://github.com/user-attachments/assets/dbe905ed-6530-4b07-aee6-752ef5545540

**16b - the static control.** Identical notes at static drive 8 - the value
the vel-coef arm reaches at full velocity, so the hard hits match 16a
exactly, while the soft hits buzz just as hard relative to their level:
static drive can't tell touch apart.

https://github.com/user-attachments/assets/4889c6f3-0112-4cbb-a27e-5cada1351569

**16c - drive on EG1.** Drive = constant 0.5 plus 3 octaves of EG1 on a
single held note - the drive swells 0.5 to 4 over four seconds, crossing the
clip knee mid-note: pure tone into growl with no parameter events after the
note-on.

https://github.com/user-attachments/assets/faa2e6ce-dafc-48b6-bfe9-8b601c88f377



## Where to look

- `src/amy.c` - `hold_and_modify` combine; `dist_process` composing a
  `dist_config_t` from authored scalars plus `msynth`, keeping
  `dist_config_t` purely the kernel's input contract.
- `src/amy.h` - the coef storage on synthinfo (authored coefs, not a
  ready-made config).
- `tests/test_dist_coefs.c` - the scale pins.

## Verified

`tests/test_dist_coefs.c` pins the scale in both directions: a velocity coef
of 2 octaves at full velocity renders identically to a stated drive of 4
(RMS 1465.0 both ways) while a coef of 1 does not (1328.4), so the check
cannot pass on wiring that ignores the coef. Mix 0 reproduces the
undistorted scene exactly; mix 1 at drive 8 does not. The scalar wire
round-trips and the stacking checks are unchanged against the parent branch,
including the single-stage RMS pin (1174.8 -> 1528.8).
