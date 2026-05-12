# Deliveries

All data deliveries are logged here. Each entry links to per-delivery notes with clip counts, QA findings, known limitations, and the SynthBanshee commit that produced the batch.

---

## Delivery 003 — multi-project, multi-voice

**Date:** 2026-05-12 · **Status:** provisional · **PR:** [#5](https://github.com/DataHackIL/avdp-synth-corpus/pull/5)

This is the current working delivery. It replaces delivery-002.

### At a glance

| | |
|---|---|
| Clips | 20 |
| Total duration | ~41.6 min |
| Projects | `she_proves` (12) + `elephant_in_the_room` (8) |
| Tiers | A (12 clean) + B (8 room-augmented) |
| TTS backends | Azure (18) + Google Chirp 3 HD (2) |
| Validation failures | 0 / 20 |
| Pipeline | SynthBanshee `0.1.0` @ [`1ea48f3`](https://github.com/DataHackIL/SynthBanshee/commit/1ea48f3) |

[Full notes](https://github.com/DataHackIL/avdp-synth-corpus/blob/main/deliveries/003-multi-project-multi-voice/notes.md) · [QA report](https://github.com/DataHackIL/avdp-synth-corpus/blob/main/deliveries/003-multi-project-multi-voice/qa-report.json)

### QA findings — closed (vs. delivery-002)

| Finding | Delivery-002 | Delivery-003 |
|---------|:---:|:---:|
| `agg_no_escalation` | 3 clips | **0** — AGG RMS now escalates with intensity |
| `warn_no_overlap` | 4 clips | **0** — overlap_ratio 100% on I4+ clips |
| `warn_emotion_downgrade` | 4 clips | **0** — emotion_downgrade_ratio 0% |
| `generation_metadata` absent | 0 of 8 clips | **20 of 20** carry the full block |
| `dirty_file_path` null | 7 of 8 clips | **20 of 20** retain dirty files |
| `normalized_dbfs` hardcoded `-1.0` | all 8 clips | **fixed** — now the measured peak |

Additional findings closed by the 2026-05-12 schema-shift regen (PRs [#110](https://github.com/DataHackIL/SynthBanshee/pull/110)/[#111](https://github.com/DataHackIL/SynthBanshee/pull/111)/[#112](https://github.com/DataHackIL/SynthBanshee/pull/112)):

| Finding | Resolution |
|---------|-----------|
| `single_backend` false positive | `qa.py` now derives backend diversity from `generation_metadata.tts_backend.values()`; reports `clips_by_tts_backend: {azure: 18, google: 2}` |
| Absolute paths in clip JSON | `dirty_file_path` and `transcript_path` are now repo-relative POSIX strings |
| Leaked pytest tmp_path on `sp_neu_a_0001_00` | Regen overwrote with canonical path; autouse env-var strip fixture prevents future leaks |

### QA findings — open

| Finding | Detail |
|---------|--------|
| `low_voice_diversity_male` | 2 voice families per gender; threshold ≥ 3 |
| `low_voice_diversity_female` | 2 voice families per gender; threshold ≥ 3 |
| `vic_f0_high` (2 clips) | `sp_sv_a_0003_00` and `sp_it_a_0003_00` — Google Chirp HD female F0 runs higher than Azure Hila reference |
| `quality_flagged_clips: 15` | Mostly from prosody cap activations at I3+; expected behaviour |

### Known limitations

- **Speaker-disjoint splits not feasible.** 4 unique speaker personas across 20 clips; all clips are `split: train`.
- **Two speaker directories only.** `agg_m_30-45_002/` and `ben_m_40-55_003/` are first appearances — code hardcoding `agg_m_30-45_001/` will miss them.
- **One room type.** All 8 Elephant Tier B clips use `clinic_office`. Future deliveries will add `welfare_office` and `open_office`.
- **Toy corpus only.** 20 clips is not sufficient for training production models.

### What this delivery exercises

1. Full `ClipMetadata` schema including `generation_metadata`, `voice_family`, and (for Tier B) the populated `acoustic_scene` block
2. Per-surface casing rules: UPPERCASE `speaker_id`, lowercase paths and clip IDs
3. `has_violence` derivation from events: NEG clips are correctly `false` even at `max_intensity ≥ 3`
4. Multi-project layout under a single `data/he/` root
5. Multi-backend provenance: `generation_metadata.tts_backend` per speaker

---

## Delivery log

| # | Date | Slug | Project | Tier | Clips | Duration | Status |
|---|------|------|---------|------|------:|------:|--------|
| [003](https://github.com/DataHackIL/avdp-synth-corpus/blob/main/deliveries/003-multi-project-multi-voice/notes.md) | 2026-05-12 | multi-project-multi-voice | she_proves + elephant | A + B | 20 | ~42m | provisional |
| [002](https://github.com/DataHackIL/avdp-synth-corpus/blob/main/deliveries/002-m2a-wettest/notes.md) | 2026-04-15 | m2a-wettest | she_proves | A | 8 | ~17m | superseded |
| [001](https://github.com/DataHackIL/avdp-synth-corpus/blob/main/deliveries/001-debug-run-1/notes.md) | 2026-04-15 | debug-run-1 | she_proves | A | 1 | 2m 36s | superseded |

## Status definitions

| Status | Meaning |
|--------|---------|
| `provisional` | Wet-test batch; not yet approved for model training |
| `approved` | QA passed; cleared for training use |
| `superseded` | Replaced by a later delivery with the same scenes at higher quality |
