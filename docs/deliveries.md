# Deliveries

What's currently in the corpus, what's missing, and what changed in the latest batch. One row per data delivery in the log at the bottom.

---

## Current delivery — 003

<span class="status-pill provisional">provisional · 2026-05-12</span> [`#5`](https://github.com/DataHackIL/avdp-synth-corpus/pull/5) · slug: `multi-project-multi-voice` · supersedes delivery-002.

### What's in it

| | |
|---|---|
| Clips | 20 |
| Total duration | ~41.6 min |
| Projects | `she_proves` (12) + `elephant_in_the_room` (8) |
| Tiers | A (12 clean) + B (8 room-augmented) |
| TTS backends | Azure (18) + Google Chirp 3 HD (2) |
| Unique speaker personas | 6 (4 in She-Proves, 2 in Elephant) |
| Validation failures | 0 / 20 |
| Pipeline | SynthBanshee `0.1.0` @ [`1ea48f3`](https://github.com/DataHackIL/SynthBanshee/commit/1ea48f3) |

Authoritative records: [`metadata.yaml`](https://github.com/DataHackIL/avdp-synth-corpus/blob/main/deliveries/003-multi-project-multi-voice/metadata.yaml) · [`notes.md`](https://github.com/DataHackIL/avdp-synth-corpus/blob/main/deliveries/003-multi-project-multi-voice/notes.md) · [`qa-report.json`](https://github.com/DataHackIL/avdp-synth-corpus/blob/main/deliveries/003-multi-project-multi-voice/qa-report.json).

### Known limitations

- **All clips are `split: train`.** Only 4 unique speaker personas across 20 clips — speaker-disjoint partitioning isn't feasible at this scale.
- **One room type for Elephant.** All 8 Tier-B clips use `clinic_office`. `welfare_office` and `open_office` are in the pipeline but not exercised yet.
- **One device profile for She-Proves.** No `phone_in_pocket` etc. augmentation applied yet — Tier-A clips are clean, not phone-captured.
- **Voice diversity is low.** 2 voice families per gender; the QA threshold for "diverse" is ≥3.
- **Toy-batch scale.** 20 clips is enough to wire up consumer plumbing. Not enough to train a production model.

### Open QA flags

| Flag | Detail | What to do about it |
|------|--------|---------------------|
| `low_voice_diversity_male` | 2 male voice families across the corpus (threshold ≥3) | Track per-voice eval separately; expect feature overfit to AvriNeural until more voices land |
| `low_voice_diversity_female` | Same, for female voices | Same |
| `vic_f0_high` (per-clip × 2) | `sp_sv_a_0003_00`, `sp_it_a_0003_00` — Google Chirp HD female F0 above Azure baseline | **Nothing.** Don't exclude the clips. Calibrate F0 features per backend if you compute them. See [Audio Format](audio-format.md#vic_f0_high-on-the-2-google-clips). |
| `quality_flagged_clips: 15` | Mostly `emotion_downgrade` from prosody cap activations at I3+ | Don't reflexively filter these out — they pass validation. See [Common mistakes #7](gotchas.md#7-quality_flags-doesnt-mean-broken). |

### Distribution

| Typology | Tier A (She-Proves) | Tier B (Elephant) | Total |
|----------|:--:|:--:|:--:|
| `SV`  | 3 | 2 | 5 |
| `IT`  | 3 | 2 | 5 |
| `NEG` | 3 | 2 | 5 |
| `NEU` | 3 | 2 | 5 |

`max_intensity` across the 20 clips: I5 = 10 clips · I3 = 4 clips · I2 = 6 clips.

---

## What this delivery exercises

Use these to check your consumer code on the schema features the delivery was designed to cover:

1. Full `ClipMetadata` schema — including the `generation_metadata` block and (for Tier B) populated `acoustic_scene`.
2. Per-surface casing rules — UPPERCASE `speaker_id`, lowercase paths and clip IDs.
3. `has_violence` derivation from events — NEG clips correctly `false` even at `max_intensity ≥ 3`.
4. Multi-project layout under a single `data/he/` root.
5. Multi-backend provenance — `generation_metadata.tts_backend` differs per speaker.

---

## What changed vs delivery-002

??? abstract "Closed QA findings (vs. delivery-002)"
    | Finding | Delivery-002 | Delivery-003 |
    |---------|:---:|:---:|
    | `agg_no_escalation` | 3 clips | **0** — AGG RMS now escalates with intensity |
    | `warn_no_overlap` | 4 clips | **0** — turn-overlap fires on I4+ clips |
    | `warn_emotion_downgrade` | 4 clips | **0** |
    | `generation_metadata` absent | 0 of 8 clips had it | **20 of 20** carry the full block |
    | `dirty_file_path` null | 7 of 8 clips | **20 of 20** retain dirty files |
    | `normalized_dbfs` hardcoded `-1.0` | all 8 clips | Records the measured peak |

??? abstract "Closed by the 2026-05-12 schema-shift regen"
    Three SynthBanshee PRs landed alongside the regen ([#110](https://github.com/DataHackIL/SynthBanshee/pull/110) / [#111](https://github.com/DataHackIL/SynthBanshee/pull/111) / [#112](https://github.com/DataHackIL/SynthBanshee/pull/112)):

    | Finding | Resolution |
    |---------|-----------|
    | `single_backend` false positive | `qa.py` derives backend diversity from `generation_metadata.tts_backend.values()`; reports `clips_by_tts_backend: {azure: 18, google: 2}` |
    | Absolute paths in clip JSON | `dirty_file_path` and `transcript_path` are now repo-relative POSIX |
    | Leaked pytest tmp_path on `sp_neu_a_0001_00` | Regen overwrote with canonical path; autouse env-var strip fixture prevents future leaks |

---

## Delivery log

| # | Date | Slug | Project | Tier | Clips | Duration | Status |
|---|------|------|---------|------|------:|------:|--------|
| [003](https://github.com/DataHackIL/avdp-synth-corpus/blob/main/deliveries/003-multi-project-multi-voice/notes.md) | 2026-05-12 | multi-project-multi-voice | she_proves + elephant | A + B | 20 | ~42m | <span class="status-pill provisional">provisional</span> |
| [002](https://github.com/DataHackIL/avdp-synth-corpus/blob/main/deliveries/002-m2a-wettest/notes.md) | 2026-04-15 | m2a-wettest | she_proves | A | 8 | ~17m | <span class="status-pill superseded">superseded</span> |
| [001](https://github.com/DataHackIL/avdp-synth-corpus/blob/main/deliveries/001-debug-run-1/notes.md) | 2026-04-15 | debug-run-1 | she_proves | A | 1 | 2m 36s | <span class="status-pill superseded">superseded</span> |

## Status definitions

| Status | Meaning |
|--------|---------|
| `provisional` | Preview batch; consumer-integration only, not approved for training |
| `approved` | QA passed; cleared for training use |
| `superseded` | Replaced by a later delivery covering the same scenes at higher quality |
