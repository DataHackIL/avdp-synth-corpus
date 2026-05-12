# Audio Format

All clips in the corpus conform to the following hard constraints. Clips that fail these checks are rejected at generation time and will not appear in the corpus.

---

## Format requirements

| Property | Value |
|----------|-------|
| Sample rate | 16 000 Hz |
| Channels | 1 (mono) |
| Bit depth | 16-bit PCM |
| Peak level | ≤ –1.0 dBFS (safety ceiling) |
| Duration | ≥ 3.0 s |
| Encoding | WAV (no lossy formats) |

```python
import soundfile as sf
import numpy as np

wav, sr = sf.read("data/he/agg_m_30-45_001/sp_sv_a_0001_00.wav")
assert sr == 16000
assert wav.ndim == 1               # mono
assert wav.dtype == np.float64     # soundfile returns float64 by default
assert np.abs(wav).max() <= 1.0   # -1.0 dBFS ≈ linear amplitude 1.0

# Check format info
info = sf.info("data/he/agg_m_30-45_001/sp_sv_a_0001_00.wav")
print(info.subtype)  # PCM_16
```

---

## Normalization pipeline

Each clip passes through two normalization steps:

```
TTS render (float32, arbitrary loudness)
    ↓
[1] Per-turn RMS gain (M3a)        — preserves inter-turn contrast
    ↓
[2] Single global peak gain         — lands absolute peak at target_peak_dbfs
    ↓
[3] Safety limiter                  — clips at ≤ –1.0 dBFS (guaranteed no-op for target ≥ –12.0)
    ↓
Tier B only: room IR + device → renormalize to same target
    ↓
Output WAV
```

### Step 1 — Per-turn RMS gain (M3a)

Each dialogue turn is gain-adjusted so its RMS matches a per-intensity target. This preserves the acoustic contrast between calm and escalated turns — a whispered turn at I1 stays quieter than a shouted turn at I5 — while giving the subsequent global normalization a stable peak-to-RMS ratio to work with.

??? info "Why per-turn RMS matters"
    Without per-turn normalization, the TTS engine produces flat RMS across intensities regardless of the requested prosody. The raw Azure and Google outputs are nearly constant-loudness even when the SSML requests "shout" style. Per-turn RMS gain is the mechanism that creates the acoustic loudness gradient you expect to see in the data.

### Step 2 — Single global peak gain

A single gain is applied to the whole mix so the clip's absolute peak lands at `loudness_target_peak_dbfs` (default: –2.0 dBFS). Because it's a single gain, all per-turn RMS *ratios* survive unchanged — the contrast from Step 1 is preserved.

The configured target is recorded in `generation_metadata.loudness_target_peak_dbfs`.
The measured output peak is recorded in `preprocessing_applied.normalized_dbfs`.

### Step 3 — Safety limiter

A hard ceiling at –1.0 dBFS. For in-spec targets (range: [–12.0, –1.5] dBFS), this is a guaranteed no-op. It exists as a safety rail against misconfiguration.

---

## Silence padding

Every clip has at least 0.5 s of ambient silence at the head and tail. This is applied by `preprocess()` and logged in `preprocessing_applied.silence_padded: true`.

Onset/offset timestamps in the `.txt` transcript and `.jsonl` events are already shifted to account for the leading pad — they refer to positions in the final processed WAV, not the raw TTS output.

---

## Dirty files

`preprocessing_applied` records the processing that was applied. The **pre-preprocessing WAV** is retained as `{clip_id}_dirty.wav` under `assets/speech/dirty/`. These are the raw TTS-mixer outputs before normalization, padding, or denoising.

The `dirty_file_path` field in ClipMetadata gives the repo-relative path:
```
"dirty_file_path": "assets/speech/dirty/sp_sv_a_0001_00_dirty.wav"
```

Dirty files are useful for:
- Diagnosing normalization issues (compare dirty peak vs. `normalized_dbfs`)
- Checking raw TTS prosody before processing
- Re-running preprocessing with different parameters

!!! warning "Do not modify dirty files"
    The `assets/` directory is managed by SynthBanshee. Manual edits to `.wav` files under `assets/speech/` will break SHA-256 cache lookups.

---

## TTS backends

| Backend | Voices | Clips in delivery-003 |
|---------|--------|----------------------|
| Azure Cognitive Services | `he-IL-AvriNeural` (M), `he-IL-HilaNeural` (F) | 18 |
| Google Cloud TTS Chirp 3 HD | `he-IL-Chirp3-HD-Achird` (M), `he-IL-Chirp3-HD-Achernar` (F) | 2 |

The backend per speaker is recorded in `generation_metadata.tts_backend`:
```json
"tts_backend": {
    "AGG_M_30-45_002": "google",
    "VIC_F_25-40_003": "google"
}
```

??? info "Azure SSML cache"
    SynthBanshee caches per-utterance WAVs under `assets/speech/` keyed by SHA-256 of the full rendered SSML string. Re-running generation with the same SSML is **free** for Azure clips — the file is returned directly from cache without an API call. Google Chirp HD does not use the same cache: it produces slightly different audio on each synthesis (minor bit-level variation at the same parameters).

---

## Known audio quirks

### `vic_f0_high` — Google Chirp HD female F0 baseline

The two Google Chirp 3 HD clips (`sp_sv_a_0003_00`, `sp_it_a_0003_00`) use the female voice `he-IL-Chirp3-HD-Achernar`. This voice's F0 baseline runs measurably higher than `he-IL-HilaNeural` (Azure), against which the corpus QA M10a thresholds were calibrated.

Both clips are flagged `vic_f0_high` in the QA report. This is expected and tracked — it reflects a real backend difference, not a synthesis failure. **Do not exclude these clips** on the basis of this flag; calibrate your model's F0 features against the correct baseline per backend.

### `quality_flags: ["emotion_downgrade"]`

Several clips carry an `emotion_downgrade` quality flag. This means the TTS engine produced a less emotionally intense output than requested by the SSML prosody hints — the pipeline detected the downgrade and flagged it. Audio quality is still acceptable; the prosody is slightly less extreme than the scene specification intended.

In delivery-003: 15 clips carry at least one quality flag, mostly from prosody cap activations at I3+.
