# Schema Reference

Every clip's `.json` file contains a `ClipMetadata` object. The authoritative Pydantic model is in [SynthBanshee `synthbanshee/labels/schema.py`](https://github.com/DataHackIL/SynthBanshee/blob/main/synthbanshee/labels/schema.py).

---

## Loading with Pydantic

```python
from synthbanshee.labels.schema import ClipMetadata  # requires SynthBanshee installed
from pathlib import Path

meta = ClipMetadata.model_validate_json(
    Path("data/he/agg_m_30-45_001/sp_sv_a_0001_00.json").read_text()
)
print(meta.clip_id, meta.violence_typology, meta.weak_label.has_violence)
# sp_sv_a_0001_00 SV True
```

Plain JSON (no SynthBanshee required):

```python
import json
from pathlib import Path

meta = json.loads(Path("data/he/agg_m_30-45_001/sp_sv_a_0001_00.json").read_text())
```

---

## Top-level `ClipMetadata` fields

| Field | Type | Description |
|-------|------|-------------|
| `clip_id` | `str` | Lowercase ASCII clip identifier, e.g. `sp_sv_a_0001_00` |
| `project` | `str` | `she_proves` or `elephant_in_the_room` |
| `language` | `str` | ISO 639-1, always `"he"` |
| `violence_typology` | `str` | `SV` / `IT` / `NEG` / `NEU` — see [taxonomy](taxonomy.md) |
| `tier` | `str` | `"A"` (clean) or `"B"` (room-augmented) |
| `duration_seconds` | `float` | Duration of the processed WAV |
| `sample_rate` | `int` | Always `16000` |
| `channels` | `int` | Always `1` |
| `is_synthetic` | `bool` | Always `true` in this corpus |
| `generator_version` | `str` | SynthBanshee semver, e.g. `"0.1.0"` |
| `generation_date` | `str` | ISO 8601 date of generation |
| `random_seed` | `int` | Scene-level RNG seed for reproducibility |
| `scene_config` | `str` | Relative path to the scene YAML in SynthBanshee |
| `transcript_path` | `str` | Repo-relative POSIX path to the `.txt` transcript |
| `dirty_file_path` | `str` | Repo-relative POSIX path to the pre-preprocessing WAV |
| `speakers` | `list[SpeakerInfo]` | Speaker metadata — see below |
| `weak_label` | `WeakLabel` | Clip-level summary labels |
| `generation_metadata` | `GenerationMetadata \| null` | Pipeline provenance — see below |
| `preprocessing_applied` | `PreprocessingApplied` | What preprocessing steps ran |
| `acoustic_scene` | `AcousticScene` | Room/device augmentation (Tier B) |
| `quality_flags` | `list[str]` | QA flags, e.g. `["emotion_downgrade"]` |
| `snr_db_estimated` | `float \| null` | Estimated SNR (not always populated) |
| `annotator_confidence` | `float` | Auto-label confidence, 0–1 (auto-generated: always `1.0`) |
| `iaa_reviewed` | `bool` | Whether inter-annotator agreement review was done |
| `she_proves_meta` | `null` | Reserved for She-Proves–specific metadata (future) |
| `elephant_meta` | `null` | Reserved for Elephant–specific metadata (future) |

---

## `SpeakerInfo`

One entry per speaker in `speakers[]`.

| Field | Type | Description |
|-------|------|-------------|
| `speaker_id` | `str` | UPPERCASE persona ID, e.g. `AGG_M_30-45_001` |
| `role` | `str` | `AGG` (aggressor), `VIC` (victim), `SW` (social worker), `BEN` (beneficiary/client) |
| `gender` | `str` | `"male"` or `"female"` |
| `age_range` | `str` | e.g. `"30-45"` |
| `tts_voice_id` | `str` | TTS voice identifier, e.g. `"he-IL-AvriNeural"` |
| `voice_family` | `str` | Same as `tts_voice_id` (may diverge in future) |

??? info "Speaker ID casing convention"
    The `speaker_id` field in JSON is always **UPPERCASE**: `AGG_M_30-45_001`.
    The on-disk directory is **lowercase**: `agg_m_30-45_001/`.
    This is a deliberate per-surface casing rule — see [SynthBanshee spec §2.5](https://github.com/DataHackIL/SynthBanshee/blob/main/docs/spec.md#25-filename-constraints).

---

## `WeakLabel`

| Field | Type | Description |
|-------|------|-------------|
| `has_violence` | `bool` | `any(e.tier1_category != "NONE" for e in events)` — see [taxonomy](taxonomy.md#has_violence-the-correct-derivation) |
| `violence_typology` | `str` | Mirrors top-level `violence_typology` |
| `max_intensity` | `int` | Highest per-turn intensity across the clip (1–5) |
| `violence_categories` | `list[str]` | Distinct `tier1_category` values observed in events |

---

## `GenerationMetadata`

Present on all delivery-003 clips; may be `null` on older clips.

| Field | Type | Description |
|-------|------|-------------|
| `pipeline_version` | `str` | SynthBanshee semver |
| `tts_backend` | `dict[str, str]` | Speaker ID → `"azure"` or `"google"` |
| `voice_family` | `dict[str, str]` | Speaker ID → voice family string |
| `mix_mode_used` | `str` | `"sequential"` (turns in order) or `"overlapping"` |
| `normalization_strategy` | `str` | `"per_turn_rms_v2_target_peak"` |
| `loudness_target_peak_dbfs` | `float` | Configured peak target, e.g. `-2.0` |
| `breathiness_applied` | `bool` | Whether breathiness augmentation was applied |
| `effective_prosody_caps` | `list[ProsodyCap]` | Per-turn cap activations at I3–I5 |
| `speaker_state_serialized` | `dict[str, SpeakerState]` | Final prosody state per speaker |
| `prosody_controller_version` | `str \| null` | Version of the prosody controller |
| `text_normalization_version` | `str \| null` | Version of text normalization |
| `timing_controller_version` | `str \| null` | Version of timing controller |

### `ProsodyCap` (entry in `effective_prosody_caps`)

| Field | Description |
|-------|-------------|
| `turn_index` | Zero-based turn index |
| `intensity` | Intensity score for that turn |
| `dim` | `"pitch"` or `"rate"` |
| `pre_cap` | Prosody value before capping (semitones for pitch, ratio for rate) |
| `post_cap` | Prosody value after capping |

### `SpeakerState` (entry in `speaker_state_serialized`)

| Field | Description |
|-------|-------------|
| `pitch_offset_st` | Final pitch offset in semitones |
| `rate_offset` | Final speaking rate multiplier |
| `volume_offset_db` | Final volume offset in dB |
| `breathiness_level` | Breathiness level 0–1 |

---

## `PreprocessingApplied`

| Field | Type | Description |
|-------|------|-------------|
| `resampled_to_16k` | `bool` | Whether sample rate conversion ran |
| `downmixed_to_mono` | `bool` | Whether channel downmix ran |
| `normalized_dbfs` | `float` | **Measured** peak dBFS of the output WAV (not the target) |
| `silence_padded` | `bool` | Whether silence padding was applied |
| `denoised` | `bool` | Whether denoising ran |
| `spectral_filtered` | `bool` | Whether spectral filtering ran |

!!! note "`normalized_dbfs` is the measured peak, not the target"
    Use `generation_metadata.loudness_target_peak_dbfs` for the configured target.
    Use `preprocessing_applied.normalized_dbfs` to verify the actual output peak.
    On delivery-003, both should be very close to `–2.0` (within floating-point precision).

---

## `AcousticScene`

Populated for Tier B clips. Null fields indicate Tier A (no augmentation).

| Field | Type | Description |
|-------|------|-------------|
| `room_type` | `str \| null` | e.g. `"clinic_office"`, `"welfare_office"`, `"open_office"` |
| `device` | `str \| null` | e.g. `"pi_budget_mic"` |
| `ir_source` | `str \| null` | Room impulse response source, e.g. `"pyroomacoustics_ism"` |
| `snr_db_actual` | `float \| null` | Actual SNR after augmentation (dB) |
| `speaker_distance_meters` | `float \| null` | Simulated speaker distance from microphone |
| `background_events` | `list[BackgroundEvent]` | Non-speech acoustic events added |

### `BackgroundEvent`

| Field | Description |
|-------|-------------|
| `type` | `"hvac_hum"`, `"ACOU_SLAM"`, `"ACOU_FALL"`, etc. |
| `onset` | Start time in seconds |
| `offset` | End time in seconds |
| `level_db` | Relative level of the event (dB) |

---

## `EventLabel` (`.jsonl` rows)

One JSON object per line. Each represents a single labelled event within the clip.

| Field | Type | Description |
|-------|------|-------------|
| `event_id` | `str` | `{clip_id}_EVT_{index:03d}` |
| `clip_id` | `str` | Parent clip ID |
| `onset` | `float` | Event start time in seconds (in the processed WAV) |
| `offset` | `float` | Event end time in seconds |
| `tier1_category` | `str` | `VERB` / `DIST` / `PHYS` / `EMOT` / `ACOU` / `NONE` |
| `tier2_subtype` | `str` | e.g. `VERB_SHOUT`, `PHYS_HARD` |
| `intensity` | `int` | Turn intensity 1–5 |
| `speaker_id` | `str` | UPPERCASE speaker persona ID |
| `speaker_role` | `str` | `AGG`, `VIC`, `SW`, `BEN` |
| `emotional_state` | `str` | e.g. `"anger"`, `"fear"`, `"desperation"`, `"neutral"` |
| `confidence` | `float` | Auto-label confidence (always `1.0` for auto-generated) |
| `label_source` | `str` | `"auto"` for all current clips |
| `iaa_reviewed` | `bool` | Always `false` in current deliveries |
| `truncated` | `bool` | Whether the event was cut short by a turn boundary |
| `notes` | `str \| null` | Annotator notes |

---

## Manifest CSV columns

`data/he/manifest.csv` — one row per clip.

| Column | Type | Notes |
|--------|------|-------|
| `clip_id` | str | Matches JSON `clip_id` |
| `project` | str | `she_proves` / `elephant_in_the_room` |
| `violence_typology` | str | `SV` / `IT` / `NEG` / `NEU` |
| `tier` | str | `A` / `B` |
| `duration_seconds` | float | |
| `speaker_ids` | str | Pipe-delimited, e.g. `AGG_M_30-45_001\|VIC_F_25-40_002` |
| `voice_families` | str | Pipe-delimited, matches `speaker_ids` order |
| `has_violence` | bool | See [taxonomy](taxonomy.md#has_violence-the-correct-derivation) |
| `max_intensity` | int | 1–5 |
| `quality_flags` | str | Comma-delimited flag list |
| `split` | str | `train` / `val` / `test` — all `train` in delivery-003 |
| `wav_path` | str | Repo-relative POSIX path |
| `strong_labels_path` | str | Repo-relative POSIX path to `.jsonl` |
