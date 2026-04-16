# Delivery 001 — debug-run-1

**Date:** 2026-04-15  **Status:** superseded  **PR:** [#1](https://github.com/DataHackIL/avdp-synth-corpus/pull/1)

## Contents

| Clip ID | Project | Typology | Tier | Duration |
|---------|---------|----------|------|------:|
| sp_it_a_0001_00 | she_proves | IT | A | 2m 36s |

Single clip generated to validate the end-to-end pipeline (Stage 1 → Stage 5) and confirm
spec-compliant output (16 kHz mono WAV, `.txt`/`.json`/`.jsonl` triplet, `validate_clip` pass).

## Pipeline version

SynthBanshee `0.1.0` / commit `d760c0e`. Pre-V3 pipeline — no per-speaker prosody config,
no Hebrew gender disambiguation, no `measure-prosody` QA.

## Known limitations

- **VIC F0 too high:** baseline ~215 Hz; rises to child-voice range (~260 Hz) at intensity 5.
  Fixed in M2a (delivery 002).
- **AGG pitch creep:** `pitch_delta_st` scaled 0→+4 st across I1→I5, audible as "helium" effect.
  Fixed in M2a (delivery 002).
- **Flat loudness:** Azure normalizes WAV output; `volume_delta_db` in SSML has no measurable
  effect. Fix pending M3 (per-turn RMS gain in SceneMixer).
- **Hebrew gender errors:** AGG→VIC address forms (e.g. `שלך`) lack niqqud gender disambiguation.
  Fixed in M1 (SynthBanshee PR #23).
- **No augmentation:** Tier A, no acoustic scene applied.

## Intended use

Pipeline smoke-test and label schema validation only. Do not use for model training.
