# Delivery 003 — multi-project, multi-voice toy corpus

**Date:** 2026-05-12  **Status:** provisional  **PR:** [#TBD](https://github.com/DataHackIL/avdp-synth-corpus/pulls)

First handoff target for the She-Proves and Elephant consumer teams. Replaces
delivery 002. Designed to give both teams enough material to spec their
training pipelines around — not for model training itself.

## Contents

**20 clips. 41.7 min total. 4 unique voice families across Azure + Google backends.**

### She-Proves Tier A — Azure (10 clips)

`data/he/agg_m_30-45_001/` — speakers `AGG_M_30-45_001` + `VIC_F_25-40_002` (Azure: `he-IL-AvriNeural`, `he-IL-HilaNeural`)

| Clip ID | Typology | Duration |
|---------|----------|------:|
| sp_it_a_0001_00 | IT | 2m 23.78s |
| sp_it_a_0002_00 | IT | 2m 19.65s |
| sp_sv_a_0001_00 | SV | 1m 50.46s |
| sp_sv_a_0002_00 | SV | 1m 32.07s |
| sp_neg_a_0001_00 | NEG | 1m 58.77s |
| sp_neg_a_0002_00 | NEG | 1m 47.81s |
| sp_neg_a_0003_00 | NEG | 2m 26.27s |
| sp_neu_a_0001_00 | NEU | 1m 59.19s |
| sp_neu_a_0002_00 | NEU | 2m 09.01s |
| sp_neu_a_0003_00 | NEU | 1m 45.07s |

### She-Proves Tier A — Google Chirp HD (2 clips)

`data/he/agg_m_30-45_002/` — speakers `AGG_M_30-45_002` + `VIC_F_25-40_003` (Google Chirp HD: `he-IL-Chirp3-HD-Achird`, `he-IL-Chirp3-HD-Achernar`)

| Clip ID | Typology | Duration |
|---------|----------|------:|
| sp_sv_a_0003_00 | SV | 1m 42.80s |
| sp_it_a_0003_00 | IT | 1m 53.86s |

### Elephant in the Room Tier B (8 clips)

`data/he/ben_m_40-55_003/` — speakers `BEN_M_40-55_003` + `SW_F_30-45_001` (Azure: `he-IL-AvriNeural`, `he-IL-HilaNeural`)

| Clip ID | Typology | Duration |
|---------|----------|------:|
| el_sv_b_0001_00 | SV | 2m 27.03s |
| el_sv_b_0002_00 | SV | 2m 18.45s |
| el_it_b_0001_00 | IT | 2m 29.99s |
| el_it_b_0002_00 | IT | 2m 31.61s |
| el_neg_b_0001_00 | NEG | 1m 53.79s |
| el_neg_b_0002_00 | NEG | 2m 54.62s |
| el_neu_b_0001_00 | NEU | 1m 56.89s |
| el_neu_b_0002_00 | NEU | 1m 19.65s |

## Pipeline version

SynthBanshee `0.1.0` / commit [`d92d61e`](https://github.com/DataHackIL/SynthBanshee/commit/d92d61e) (tip of `main` at delivery time). Carries four corrections vs delivery 002:

- **[PR #102](https://github.com/DataHackIL/SynthBanshee/pull/102)** — `preprocessing_applied.normalized_dbfs` now records the *measured* post-preprocess peak (was hardcoded `-1.0`). Pair with `generation_metadata.loudness_target_peak_dbfs` to diagnose loudness drift; the schema docstring at `labels/schema.py:175` pins the measured-vs-target split.
- **[PR #103](https://github.com/DataHackIL/SynthBanshee/pull/103)** — `docs/spec.md` pinned the `has_violence` derivation rule (`any(e.tier1_category != "NONE")`), added the §2.5 identifier-casing table, rewrote §5.1 field notes.
- **[PR #105](https://github.com/DataHackIL/SynthBanshee/pull/105)** — added `sp_sv_a_0003` + `sp_it_a_0003` Google-pair shadow scenes (this delivery's voice-diversity vehicle).
- **[PR #106](https://github.com/DataHackIL/SynthBanshee/pull/106)** — root cause for [#72](https://github.com/DataHackIL/SynthBanshee/issues/72) found and fixed: `_HINT_DEFAULTS["stress"]` was emitting nested `<prosody volume="+NdB">` inside outer `<prosody volume="+N%">`, which Azure rejects with `SSML parse error 0x80045003`. **Required to unblock this delivery** — without the fix, 6 of 8 elephant Tier B scenes (every one whose LLM script carries a `stress` phrase hint at I3+) failed reliably.

## Speaker / voice / backend matrix

| Project | Speaker dir | Speakers (canonical UPPERCASE id) | Voice family — M | Voice family — F | Backend |
|---|---|---|---|---|---|
| she_proves | `agg_m_30-45_001/` | `AGG_M_30-45_001`, `VIC_F_25-40_002` | `he-IL-AvriNeural` | `he-IL-HilaNeural` | Azure |
| she_proves | `agg_m_30-45_002/` | `AGG_M_30-45_002`, `VIC_F_25-40_003` | `he-IL-Chirp3-HD-Achird` | `he-IL-Chirp3-HD-Achernar` | Google |
| elephant_in_the_room | `ben_m_40-55_003/` | `BEN_M_40-55_003`, `SW_F_30-45_001` | `he-IL-AvriNeural` | `he-IL-HilaNeural` | Azure |

This is the first delivery with multi-project and multi-backend coverage in one batch. Consumer-side teams should anchor their schema parsers on this matrix: same dataset, different `speakers[]`, different `generation_metadata.tts_backend` per clip, different `generation_metadata.voice_family` per clip.

## QA findings vs delivery 002

`synthbanshee qa-report --run-summary` over `data/he/` — failure rate **0.0%** (0 of 20 clips invalid). Full output: [`qa-report.json`](qa-report.json).

**Closed in this delivery (delivery 002 → 003):**

| Finding | Delivery 002 | Delivery 003 |
|---|---|---|
| `agg_no_escalation` | fired on 3 clips | **0** — AGG RMS now escalates with intensity (post-M3) |
| `warn_no_overlap` | fired on 4 clips | **0** — overlap_ratio 100% on I4+ clips (post-M8a) |
| `warn_emotion_downgrade` | fired on 4 clips | **0** — emotion_downgrade_ratio 0% |
| `generation_metadata` absent | 0 of 8 clips had it | **20 of 20 clips** carry the full block |
| `dirty_file_path` null | 7 of 8 clips | **20 of 20 clips** retain dirty files |
| `normalized_dbfs` hardcoded `-1.0` | all 8 clips | **fixed (#102)** — now the measured peak |

**Voice diversity — partial progress:** 1 → 2 unique voice families per gender. The `low_voice_diversity_*` thresholds expect ≥3, so the run-level warnings still fire; consumer teams should read this as "ladder climbed, not yet at the top."

**Still open** (not this delivery's scope):

- `low_voice_diversity_male` / `low_voice_diversity_female` — at 2 voices/gender; threshold is ≥3.
- `single_backend` (run-level) — **misleading**: the corpus actually uses Azure + Google. The qa-report counts `clip.tts_engine` which is currently hardcoded to `"azure_he_IL"` in `cli.py:_run_generate_pipeline`; this is a synthbanshee labeling bug, not a real diversity finding. (Tracked in a follow-up issue — see `qa-report.json` for the raw counts and `speakers[].voice_family` per clip for the actual backend distribution.)
- `vic_f0_high` (per-clip): 2 clips — `sp_it_a_0003_00` and `sp_sv_a_0003_00` — flagged. Both are the Google Chirp HD female voice (`he-IL-Chirp3-HD-Achernar`), whose F0 baseline runs higher than the Azure Hila reference the M10a thresholds were calibrated against.
- `quality_flagged_clips: 15` (mostly from `prosody_cap_activations`) — the #87 effective-prosody cap fires often at I3+; expected behaviour, recorded in `generation_metadata.effective_prosody_caps` per turn.
- General Hebrew TTS naturalness backlog ([#92](https://github.com/DataHackIL/SynthBanshee/issues/92)).

## Known limitations

- **Speaker-disjoint splits not feasible at this scale.** 4 distinct speakers across 20 clips; all clips manifest as `split: train`. Re-stratify when scaling.
- **`agg_m_30-45_002/` and `ben_m_40-55_003/`** are first appearances of those speaker dirs in the corpus — downstream tooling that hardcoded `data/he/agg_m_30-45_001/` will not find these clips.
- **Toy corpus only** — not approved for training. The point is to bootstrap consumer-side spec compliance, not to provide labelled examples in volume.

## Intended use

Spec validation and consumer-side schema bootstrapping. Concretely, this delivery exercises:

1. Full `ClipMetadata` schema including `generation_metadata`, `voice_family`, and (for Tier B) the populated `acoustic_scene` block (`room_type`, `device`, `ir_source`, `snr_db_actual`).
2. The §2.5 per-surface identifier casing rules: UPPERCASE `speaker_id` values, lowercase directory paths, lowercase clip-id filename stems.
3. The `has_violence` events-based derivation rule: NEG clips are correctly `has_violence: false` even at `max_intensity ≥ 3`.
4. Multi-project layout under a single `data/he/` root, with `project: she_proves` and `project: elephant_in_the_room` distinguishable from `.json` metadata alone.
5. Multi-backend TTS provenance via `generation_metadata.tts_backend` per speaker (`{ "AGG_M_30-45_002": "google", ... }`).

Do **not** train on this. Use it to write `ClipMetadata` parsers, manifest loaders, taxonomy validators, and ID-casing utilities that the next data delivery will not silently break.
