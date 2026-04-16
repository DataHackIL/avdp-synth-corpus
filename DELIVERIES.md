# Delivery Log

One row per data delivery (merged PR). Each entry links to a per-delivery notes file with
clip counts, prosody QA results, known limitations, and the SynthBanshee commit that produced it.

| # | Date | Slug | Project | Tier | Clips | Duration | Typologies | Pipeline milestone | Status | PR |
|---|------|------|---------|------|------:|------:|------------|-------------------|--------|-----|
| 001 | 2026-04-15 | [debug-run-1](deliveries/001-debug-run-1/notes.md) | she_proves | A | 1 | 2m 36s | IT | v1 baseline (pre-V3) | superseded | [#1](https://github.com/DataHackIL/avdp-synth-corpus/pull/1) |
| 002 | 2026-04-15 | [m2a-wettest](deliveries/002-m2a-wettest/notes.md) | she_proves | A | 8 | ~17m | SV, IT, NEG, NEU | M2a SSML prosody | provisional | [#2](https://github.com/DataHackIL/avdp-synth-corpus/pull/2) |

## Status definitions

| Status | Meaning |
|--------|---------|
| `provisional` | Wet-test batch; not yet approved for model training |
| `approved` | QA passed; cleared for training use |
| `superseded` | Replaced by a later delivery with the same scenes at higher quality |
