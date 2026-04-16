# Delivery 002 — m2a-wettest

**Date:** 2026-04-15  **Status:** provisional  **PR:** [#2](https://github.com/DataHackIL/avdp-synth-corpus/pull/2)

## Contents

| Clip ID | Project | Typology | Tier | Duration |
|---------|---------|----------|------|------:|
| sp_sv_a_0001_00 | she_proves | SV | A | 2m 02s |
| sp_sv_a_0002_00 | she_proves | SV | A | 1m 39s |
| sp_it_a_0001_00 | she_proves | IT | A | 2m 36s |
| sp_it_a_0002_00 | she_proves | IT | A | 2m 27s |
| sp_neg_a_0001_00 | she_proves | NEG | A | 2m 08s |
| sp_neg_a_0002_00 | she_proves | NEG | A | 1m 54s |
| sp_neu_a_0001_00 | she_proves | NEU | A | 2m 01s |
| sp_neu_a_0002_00 | she_proves | NEU | A | 2m 13s |

**Total:** 8 clips, ~17 minutes across all 4 violence typologies.

## Pipeline version

SynthBanshee `0.1.0` / commit `adf4a22`. M2a milestone — SSML prosody defaults redesigned
per §4.2a of `docs/audio_generation_v3_design.md`. Includes M1 Hebrew gender disambiguation
(SynthBanshee PR #23).

## Prosody QA (`synthbanshee measure-prosody`)

| Check | Value | Threshold | Result |
|-------|------:|-----------|--------|
| VIC I1 median F0 | 160.1 Hz | ≤ 200 Hz | ✅ pass |
| VIC I4 median F0 | 185.0 Hz | < 250 Hz | ✅ pass |
| VIC I5 median F0 | 200.1 Hz | < 250 Hz | ✅ pass |
| AGG I5 − I1 RMS | +0.4 dB | ≥ 8 dB | ❌ fail |

AGG RMS failure is a known Azure normalization issue — the TTS engine normalizes WAV output
regardless of the SSML `volume` attribute. Fix is gated on M3 (per-turn RMS gain applied
post-TTS inside SceneMixer).

## Subjective QA (human ear)

| Aspect | Result | Notes |
|--------|--------|-------|
| NEU pitch | ✅ pass | Calm, flat; no audible tension |
| NEG delivery | ✅ pass | Friction/argument audible |
| AGG helium artefact | ✅ pass | Pitch creep fixed; rate carries urgency |
| VIC pitch direction | ✅ pass | Flat/falling under stress; no upward creep |

## What changed vs delivery 001

- **VIC prosody (M2a):** `pitch_delta_st` inverted from rising (0 → +7 st across I1–I5)
  to capped falling (−4 → −1 st). Confirmed by `measure-prosody` and human ear.
- **AGG prosody (M2a):** `pitch_delta_st` flattened from 0→+4 st to 0/0/0/+1/+1 across
  I1–I5. Audible helium artefact eliminated; speech rate (×1.0 → ×1.25) now the primary
  urgency cue.
- **Hebrew gender disambiguation (M1):** AGG→VIC address forms now carry correct gender
  niqqud (SynthBanshee PR #23).
- **Output routing:** clips land directly in corpus repo via `SYNTHBANSHEE_DATA_DIR`.

## Known limitations

- **AGG loudness flat:** +0.4 dB RMS delta vs ≥8 dB target. Pending M3.
- **Robotic intonation / slow delivery:** TTS limitation; not targeted until later milestones.
- **No interruption / overlap:** Turns are sequential; realistic cross-talk pending.
- **No acoustic augmentation:** Tier A only; Tier B scenes (with room IR + noise) pending Phase 2.

## Intended use

Prosody QA validation for the M2a milestone. Not approved for model training until
AGG loudness escalation is resolved (M3) and a full Phase 1 batch is generated.
