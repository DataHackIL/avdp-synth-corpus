# Schema Reference

A real `ClipMetadata` JSON, fully annotated. Click the `+` markers to jump to a field's explanation. Fields are ordered by how often you'll actually use them: top-level → labels → speakers → augmentation (Tier B only) → provenance (diagnostic, usually skip).

The authoritative Pydantic model lives in [SynthBanshee `synthbanshee/labels/schema.py`](https://github.com/DataHackIL/SynthBanshee/blob/main/synthbanshee/labels/schema.py). For day-to-day consumer work, `json.loads()` is fine.

---

## Annotated example

```json
{
  "clip_id": "sp_sv_a_0001_00",                       // (1)!
  "project": "she_proves",                            // (2)!
  "language": "he",                                   // (3)!
  "violence_typology": "SV",                          // (4)!
  "tier": "A",                                        // (5)!
  "duration_seconds": 110.46,                         // (6)!
  "sample_rate": 16000,                               // (7)!
  "channels": 1,
  "is_synthetic": true,                               // (8)!

  "weak_label": {                                     // (9)!
    "has_violence": true,
    "violence_typology": "SV",
    "max_intensity": 5,
    "violence_categories": ["DIST", "PHYS", "VERB"]
  },

  "speakers": [                                       // (10)!
    {
      "speaker_id": "AGG_M_30-45_001",
      "role": "AGG",
      "gender": "male",
      "age_range": "30-45",
      "tts_voice_id": "he-IL-AvriNeural",
      "voice_family": "he-IL-AvriNeural"
    },
    {
      "speaker_id": "VIC_F_25-40_002",
      "role": "VIC",
      "gender": "female",
      "age_range": "25-40",
      "tts_voice_id": "he-IL-HilaNeural",
      "voice_family": "he-IL-HilaNeural"
    }
  ],

  "transcript_path":  "data/he/agg_m_30-45_001/sp_sv_a_0001_00.txt",        // (11)!
  "dirty_file_path":  "assets/speech/dirty/sp_sv_a_0001_00_dirty.wav",      // (12)!

  "quality_flags": ["emotion_downgrade"],             // (13)!

  "acoustic_scene": {                                 // (14)!
    "room_type": null,
    "device": null,
    "ir_source": null,
    "snr_db_actual": null,
    "speaker_distance_meters": null,
    "background_events": []
  },

  "preprocessing_applied": {                          // (15)!
    "resampled_to_16k":    true,
    "downmixed_to_mono":   true,
    "normalized_dbfs":    -2.0000002,
    "silence_padded":      true,
    "denoised":            true,
    "spectral_filtered":   true
  },

  "generation_metadata": { /* ...see below... */ },   // (16)!

  "generator_version": "0.1.0",                       // (17)!
  "generation_date":   "2026-05-12",
  "random_seed":       1201,
  "scene_config":      "configs/scenes/she_proves/sp_sv_a_0001.yaml",
  "snr_db_estimated":  null,                          // (18)!
  "annotator_confidence": 1.0,                        // (19)!
  "iaa_reviewed":         false,
  "she_proves_meta":      null,                       // (20)!
  "elephant_meta":        null
}
```

1.  Lowercase ASCII clip identifier. Pattern: `{project_prefix}_{typology}_{tier}_{scene_num}_{take}`.
2.  `she_proves` or `elephant_in_the_room`. Determines clip-id prefix (`sp_*` / `el_*`) and which `*_meta` field is non-null.
3.  ISO 639-1 — always `"he"` in this corpus.
4.  `SV` · `IT` · `NEG` · `NEU`. **Not** an ordered scale — see [Label Taxonomy](taxonomy.md). `NEG` is the hard-negative class (sounds intense, not violent).
5.  `"A"` (clean, TTS only) or `"B"` (room IR + device profile + background noise applied). Determines whether `acoustic_scene` is populated.
6.  Duration of the final processed WAV, **including** the 0.5 s silence pad on each end.
7.  Always 16000. Channels always 1. Format always 16-bit PCM WAV.
8.  Always `true` in this corpus. The field exists because future real-recording deliveries will set it `false`.
9.  Clip-level summary labels. `has_violence` is derived from events: `any(e.tier1_category != "NONE")`. Don't derive it from typology — see [Gotcha #1](gotchas.md#1-dont-derive-has_violence-from-typology).
10. One entry per speaker. The on-disk directory is **`speakers[0].speaker_id.lower()`** — UPPERCASE in JSON, lowercase on disk ([Gotcha #4](gotchas.md#4-uppercase-in-json-lowercase-on-disk)).
11. Repo-relative POSIX path to the `.txt` transcript.
12. Repo-relative POSIX path to the pre-preprocessing ("dirty") WAV, retained per spec. Useful for diagnosing normalization issues. **Don't modify** — `assets/` is managed by SynthBanshee ([Gotcha #7](gotchas.md#7-quality_flags-doesnt-mean-broken)).
13. Soft warnings. Don't filter on these reflexively — they don't fail validation. Most common: `emotion_downgrade` (TTS produced slightly less intense prosody than requested), `vic_f0_high` (Google female F0 above Azure baseline; expected on the 2 Google clips).
14. Populated for Tier B (Elephant) clips; all `null` / empty for Tier A. See [Elephant guide](elephant.md#the-acoustic_scene-field).
15. Records *what* preprocessing ran. `normalized_dbfs` is the **measured** post-preprocess peak — pair with `generation_metadata.loudness_target_peak_dbfs` (the configured target) to diagnose loudness drift.
16. Pipeline provenance. Always present on delivery-003 clips; may be `null` on older clips. Expanded below.
17. SynthBanshee version that produced this clip. Combined with `random_seed` + `scene_config`, scenes are reproducible.
18. Estimated SNR — not populated for any current delivery. Use `acoustic_scene.snr_db_actual` for Tier B.
19. Auto-label confidence; always `1.0` because labels are generated by the pipeline (not human-annotated). `iaa_reviewed` is always `false` for the same reason.
20. Reserved for per-project metadata. Always `null` in current deliveries.

---

## `generation_metadata` — pipeline provenance

Expanded view of field (16). Use this block for diagnostics, not for filtering training data.

```json
{
  "pipeline_version":      "0.1.0",
  "tts_backend":           {"AGG_M_30-45_001": "azure",  "VIC_F_25-40_002": "azure"},
  "voice_family":          {"AGG_M_30-45_001": "he-IL-AvriNeural", "VIC_F_25-40_002": "he-IL-HilaNeural"},
  "mix_mode_used":         "sequential",
  "normalization_strategy":  "per_turn_rms_v2_target_peak",   // internal version string; informational
  "loudness_target_peak_dbfs": -2.0,
  "breathiness_applied":   false,
  "effective_prosody_caps": [                                  // per-turn cap activations at I3+
    {"turn_index": 1, "intensity": 2, "dim": "rate",  "pre_cap": 0.912, "post_cap": 0.95},
    {"turn_index": 4, "intensity": 4, "dim": "pitch", "pre_cap": 2.348, "post_cap": 2.0}
  ],
  "speaker_state_serialized": {
    "AGG_M_30-45_001": {"pitch_offset_st": 1.40, "rate_offset": 1.14, "volume_offset_db":  3.80, "breathiness_level": 0.0},
    "VIC_F_25-40_002": {"pitch_offset_st": 0.56, "rate_offset": 0.89, "volume_offset_db": -2.58, "breathiness_level": 0.0}
  }
}
```

| Field | What it tells you |
|-------|-------------------|
| `tts_backend` | Per-speaker dict mapping speaker_id → `"azure"` or `"google"`. The corpus-level backend distribution is derived from this — don't look for a top-level `tts_engine` field, it was removed. |
| `voice_family` | Per-speaker dict mapping speaker_id → voice ID. Currently identical to `speakers[].tts_voice_id`. |
| `mix_mode_used` | `"sequential"` (turns in order) or `"overlapping"` (turns can overlap at I4+). All delivery-003 violent clips use `"overlapping"` at high intensity; calm clips use `"sequential"`. |
| `loudness_target_peak_dbfs` | The **configured** peak target (–2.0 dBFS by default). Pair with `preprocessing_applied.normalized_dbfs` (the measured peak) to detect drift. |
| `effective_prosody_caps` | Per-turn list of cap activations — when the LLM-suggested pitch or rate exceeded the safety cap. Common at I3+ in this delivery. Recording them lets you compute the "uncapped" prosody the LLM intended. |
| `speaker_state_serialized` | Final per-speaker prosody offset. Used for reproducing a scene with the same speaker drift. |

??? info "Internal version-string fields"
    `normalization_strategy`, `prosody_controller_version`, `text_normalization_version`, `timing_controller_version` are internal version strings. They're recorded for provenance but you won't filter on them as a consumer.

---

## `EventLabel` — `.jsonl` rows

One JSON object per line. One line per labelled event. Read line-by-line — `json.loads()` on the whole file errors.

```json
{
  "event_id":       "sp_sv_a_0001_00_EVT_004",
  "clip_id":        "sp_sv_a_0001_00",
  "onset":          36.736,
  "offset":         46.552,
  "tier1_category": "DIST",
  "tier2_subtype":  "DIST_SCREAM",
  "intensity":      4,
  "speaker_id":     "AGG_M_30-45_001",
  "speaker_role":   "AGG",
  "emotional_state": "anger",
  "confidence":      1.0,
  "label_source":    "auto",
  "iaa_reviewed":    false,
  "truncated":       false,
  "notes":           null
}
```

| Field | Notes |
|-------|-------|
| `onset` / `offset` | Seconds in the **final processed WAV**. Already shifted to account for the 0.5 s leading silence pad. |
| `tier1_category` | `VERB` · `DIST` · `PHYS` · `EMOT` · `ACOU` · `NONE`. See [Label Taxonomy](taxonomy.md). |
| `tier2_subtype`  | e.g. `VERB_SHOUT`, `DIST_SCREAM`, `PHYS_HARD`, `ACOU_SLAM`. |
| `intensity`      | The intensity of the turn the event belongs to (1–5). |
| `speaker_id`     | UPPERCASE. Matches one of `ClipMetadata.speakers[].speaker_id`. |
| `speaker_role`   | `AGG` · `VIC` · `SW` · `BEN`. See [Glossary](glossary.md#speaker-roles). |
| `emotional_state` | Free-text label of speaker emotion at this turn (e.g. `"anger"`, `"fear"`, `"desperation"`, `"neutral"`). |
| `confidence`     | Always `1.0` (labels are auto-generated). |
| `label_source`   | Always `"auto"`. |
| `iaa_reviewed`   | Always `false` in current deliveries — no human inter-annotator agreement review yet. |
| `truncated`      | `true` if the event was cut short by a turn boundary. |

---

## Manifest CSV columns

`data/he/manifest.csv` — one row per clip, the fastest entry point for filtering.

| Column | Type | Notes |
|--------|------|-------|
| `clip_id` | str | Matches `ClipMetadata.clip_id` |
| `project` | str | `she_proves` / `elephant_in_the_room` |
| `violence_typology` | str | `SV` / `IT` / `NEG` / `NEU` |
| `tier` | str | `A` / `B` |
| `duration_seconds` | float | Final WAV duration including pads |
| `speaker_ids` | str | **Pipe-delimited.** `AGG_M_30-45_001\|VIC_F_25-40_002` |
| `voice_families` | str | **Pipe-delimited**, same order as `speaker_ids` |
| `has_violence` | bool | Derived from events — see [Gotcha #1](gotchas.md#1-dont-derive-has_violence-from-typology) |
| `max_intensity` | int | 1–5 |
| `quality_flags` | str | Comma-delimited soft warnings |
| `split` | str | `train` / `val` / `test` — **all `train`** in delivery-003 ([Gotcha #6](gotchas.md#6-all-clips-are-split-train-in-delivery-003)) |
| `wav_path` | str | Repo-relative POSIX path to the `.wav` |
| `strong_labels_path` | str | Repo-relative POSIX path to the `.jsonl` |

---

## Transcript file format (`.txt`)

Plain UTF-8. One turn = one header line + one or more text lines + one action line. Hebrew text only; no Latin script in the body.

```
[CLIP_ID: sp_sv_a_0001_00]
[SPEAKER: AGG_M_30-45_001 | ROLE: AGG | ONSET: 0.76 | OFFSET: 10.07]
מה זה הארוחה הזאת? שאלתי אותך דבר אחד פשוט, לעשות ארוחת ערב נורמלית.
[ACTION: VERB_SHOUT | INTENSITY: 2]
[SPEAKER: VIC_F_25-40_002 | ROLE: VIC | ONSET: 10.49 | OFFSET: 18.74]
עבדתי עד שש היום. עשיתי מה שהספקתי...
[ACTION: VERB_SHOUT | INTENSITY: 2]
```

- The first line is a single `[CLIP_ID: ...]` header.
- Each subsequent turn is a `[SPEAKER: ... | ROLE: ... | ONSET: ... | OFFSET: ...]` line, the Hebrew text, then `[ACTION: <tier2_subtype> | INTENSITY: 1–5]`.
- `ONSET` / `OFFSET` are in seconds, relative to the final processed WAV (already include the leading pad).
- The `.jsonl` strong labels are the canonical source for events; the transcript is for human reading and as an ASR reference.
