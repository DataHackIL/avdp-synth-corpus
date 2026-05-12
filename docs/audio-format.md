# Audio Format

The three facts you need to use the data, then optional detail on how it gets that way.

---

## What you need to know

| Fact | Value | Why it matters |
|------|-------|----------------|
| **Sample rate** | 16 000 Hz | Always. Resample your features for this. |
| **Channels / depth** | mono / 16-bit PCM WAV | `wav.ndim == 1`. No lossy formats anywhere. |
| **Peak level** | ≤ –1.0 dBFS (target –2.0 dBFS) | `np.abs(wav).max() ≈ 0.79`, **not** 1.0. |
| **Silence pad** | ≥ 0.5 s at head and tail | Onset/offset timestamps **already account for it** — no shift needed. |
| **Duration** | ≥ 3.0 s | Hard minimum; clips below it are rejected. |

```python
import soundfile as sf, numpy as np

wav, sr = sf.read("data/he/agg_m_30-45_001/sp_sv_a_0001_00.wav")
assert sr == 16000
assert wav.ndim == 1
assert wav.dtype == np.float64           # soundfile default
assert np.abs(wav).max() <= 1.0          # safety ceiling at -1.0 dBFS
print(sf.info("data/he/agg_m_30-45_001/sp_sv_a_0001_00.wav").subtype)   # PCM_16
```

---

## Two peak fields, two meanings

Every clip records two related loudness values:

| Field | Set by | What it is |
|-------|--------|------------|
| `generation_metadata.loudness_target_peak_dbfs` | The pipeline config | **Configured** peak target (default –2.0 dBFS) |
| `preprocessing_applied.normalized_dbfs` | Measurement at write time | **Measured** post-preprocess peak of the actual WAV |

If those two disagree by more than a fraction of a dB, something is wrong with normalization. Useful as a diagnostic check.

---

## Known audio quirks

### `vic_f0_high` on the 2 Google clips

`sp_sv_a_0003_00` and `sp_it_a_0003_00` use the Google Chirp 3 HD female voice (`he-IL-Chirp3-HD-Achernar`). Its F0 baseline runs measurably higher than the Azure reference voice (`he-IL-HilaNeural`), against which the QA F0 thresholds were calibrated.

**What to do about it:** nothing. The flag fires correctly; the audio is fine. If you compute F0-derived features, calibrate per backend (`generation_metadata.tts_backend`) — or just use spectral features that aren't sensitive to baseline F0. Don't exclude these two clips: they're the only backend diversity you have in this delivery.

### `quality_flags: ["emotion_downgrade"]`

The pipeline detected that the TTS engine produced slightly less intense prosody than the SSML asked for at high-intensity turns. The audio is still valid; the prosody is just a touch tamer than the scene intended. About 15 of 20 clips in delivery-003 carry this flag — it's not a defect signal.

### Dirty files

The pre-preprocessing WAV is retained at `assets/speech/dirty/{clip_id}_dirty.wav`. Its path is recorded in `dirty_file_path`. These files are the raw TTS-mixer outputs before normalization, padding, or denoising — useful for diagnosing the pipeline, not for training.

!!! warning "Don't modify files under `assets/`"
    `assets/speech/` is the SynthBanshee SHA-256 SSML cache. Renaming or editing any file there will break cache lookups and force a paid re-synthesis on next run.

---

## TTS backends

| Backend | Voices in delivery-003 | Clips |
|---------|-----------------------|------:|
| Azure Cognitive Services | `he-IL-AvriNeural` (M), `he-IL-HilaNeural` (F) | 18 |
| Google Cloud TTS Chirp 3 HD | `he-IL-Chirp3-HD-Achird` (M), `he-IL-Chirp3-HD-Achernar` (F) | 2 |

Per-speaker backend is in `generation_metadata.tts_backend`:

```json
"tts_backend": {
    "AGG_M_30-45_002": "google",
    "VIC_F_25-40_003": "google"
}
```

Azure is deterministic — re-rendering the same SSML returns byte-identical WAVs (via the SHA-256 cache). Google Chirp 3 HD is not — it produces minor bit-level variation on each synthesis at the same parameters. If you need byte-stable reproducibility for an experiment, you may see the Google clips re-render slightly differently between fresh generations even though peak / RMS / duration stay within tolerance.

---

## How the normalization actually works

You don't need this to consume the data. Open the section below if you're debugging loudness drift, building a comparable pipeline, or just curious.

??? info "The normalization pipeline (3 stages)"
    ```
    TTS render  →  per-turn RMS gain  →  single global peak gain  →  safety limiter  →  Tier B: room IR + device + noise → renormalize  →  output WAV
    ```

    **Stage 1: per-turn RMS gain.** Each dialogue turn is gain-adjusted so its RMS matches a per-intensity target. This creates the calm-to-loud gradient you'd expect — a whispered I1 turn stays quieter than a shouted I5 turn. Without this step, raw Azure and Google output is nearly constant-loudness regardless of the requested prosody style.

    **Stage 2: single global peak gain.** A single multiplicative gain lands the clip's absolute peak at `loudness_target_peak_dbfs` (default –2.0 dBFS). Because it's one gain applied to the whole mix, every per-turn RMS ratio from Stage 1 survives unchanged.

    **Stage 3: safety limiter.** A hard ceiling at –1.0 dBFS. For in-spec targets in `[-12.0, -1.5]` dBFS, this is always a no-op. It exists as a safety rail against config drift.

    **Tier B post-processing.** Room IR convolution, device frequency response (e.g. `pi_budget_mic`), and background-noise injection happen after Stage 3. Then the same `peak_normalize_to_target` helper renormalises so every tier exits at the same absolute peak — Tier A and Tier B are comparable on the loudness dimension.

??? info "Why per-turn RMS gain matters"
    Without it, the TTS engine produces flat RMS across turns regardless of the requested prosody. The raw Azure and Google outputs are nearly constant-loudness even when the SSML requests a "shout" style or sets `prosody volume="+50%"`. Per-turn RMS gain is the mechanism that creates the acoustic loudness gradient between calm and escalated turns — without it, your model has nothing to learn loudness escalation from.

??? info "Why peak normalize to –2.0 dBFS instead of 0 dBFS"
    The 2 dB of headroom buys safety against any later processing step that might add 1–2 dB of gain (room IR convolution can do this). Peak at 0 dBFS would clip; peak at –1.0 dBFS leaves no headroom for the limiter. –2.0 is the conservative middle.
