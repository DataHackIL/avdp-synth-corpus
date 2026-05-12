# She-Proves Team Guide

She-Proves is a smartphone app that **passively monitors audio for domestic violence incidents** and preserves evidence for legal use.

**Optimization target: high recall.** It is better to flag an incident for review than to miss one.

---

## Scene structure

| Property | Value |
|----------|-------|
| Duration | 3–6 minutes |
| Tier | A (clean — no room processing) |
| Pre-incident window | ≥ 60% of clip duration before the first violence event |
| Device profile | `phone_in_pocket`, `phone_on_table`, `phone_in_hand` |
| Room types | apartment rooms (living room, bedroom, kitchen) |
| Language | Hebrew (`he`) |

The long pre-incident window reflects real-world deployment: the app is always listening, and incidents are rare. Models trained on this data should handle extended periods of mundane speech before a rapid escalation.

??? info "Tier A — what does 'clean' mean?"
    Tier A clips have **no acoustic augmentation** — no room impulse response convolution, no device frequency response, no background noise injection. The audio is the direct TTS-mixer output after preprocessing: peak-normalized, silence-padded, 16 kHz mono 16-bit PCM.

    For Tier A, `acoustic_scene.room_type`, `device`, `ir_source`, and `snr_db_actual` are all `null`.

    Tier B (used by Elephant) adds all of the above. See [Elephant in the Room](elephant.md) for details.

---

## Speaker pairs

Delivery-003 has two She-Proves speaker pairs — one per TTS backend.

| Pair | Speaker dir | Male speaker | Female speaker | Backend |
|------|-------------|--------------|----------------|---------|
| Azure | `agg_m_30-45_001/` | `AGG_M_30-45_001` → `he-IL-AvriNeural` | `VIC_F_25-40_002` → `he-IL-HilaNeural` | Azure |
| Google Chirp HD | `agg_m_30-45_002/` | `AGG_M_30-45_002` → `he-IL-Chirp3-HD-Achird` | `VIC_F_25-40_003` → `he-IL-Chirp3-HD-Achernar` | Google |

Both pairs play **AGG (aggressor, male) + VIC (victim, female)** roles. The Google pair was added in delivery-003 specifically to introduce backend diversity.

!!! note "Two speaker directories"
    Clips from the Azure pair live under `data/he/agg_m_30-45_001/`.
    Clips from the Google pair live under `data/he/agg_m_30-45_002/`.
    Downstream code that hardcodes `agg_m_30-45_001/` will miss the Google clips.
    Use `manifest.csv` or filter `meta["generation_metadata"]["tts_backend"]` to find both.

---

## Clips in delivery-003

### Azure pair — 10 clips

`data/he/agg_m_30-45_001/`

| Clip ID | Typology | `has_violence` | Duration |
|---------|----------|:---:|------:|
| `sp_sv_a_0001_00` | SV | ✓ | 1m 50.5s |
| `sp_sv_a_0002_00` | SV | ✓ | 1m 32.1s |
| `sp_it_a_0001_00` | IT | ✓ | 2m 23.8s |
| `sp_it_a_0002_00` | IT | ✓ | 2m 19.7s |
| `sp_neg_a_0001_00` | NEG | — | 1m 58.8s |
| `sp_neg_a_0002_00` | NEG | — | 1m 47.8s |
| `sp_neg_a_0003_00` | NEG | — | 2m 26.3s |
| `sp_neu_a_0001_00` | NEU | — | 1m 59.2s |
| `sp_neu_a_0002_00` | NEU | — | 2m 09.0s |
| `sp_neu_a_0003_00` | NEU | — | 1m 45.1s |

### Google Chirp HD pair — 2 clips

`data/he/agg_m_30-45_002/`

| Clip ID | Typology | `has_violence` | Duration | Note |
|---------|----------|:---:|------:|------|
| `sp_sv_a_0003_00` | SV | ✓ | 1m 42.8s | `vic_f0_high` flag |
| `sp_it_a_0003_00` | IT | ✓ | 1m 53.9s | `vic_f0_high` flag |

The `vic_f0_high` flag on the Google clips indicates the female voice (`he-IL-Chirp3-HD-Achernar`) has a higher F0 baseline than the Azure Hila reference. See [Audio Format → vic_f0_high](audio-format.md#vic_f0_high-google-chirp-hd-female-f0-baseline).

---

## Loading She-Proves clips

```python
import json
import soundfile as sf
import pandas as pd
from pathlib import Path

root = Path(".")

# Via manifest — easiest
df = pd.read_csv("data/he/manifest.csv")
sp_clips = df[df["project"] == "she_proves"]

# Load all She-Proves audio
wavs = {}
for _, row in sp_clips.iterrows():
    wav, sr = sf.read(root / row["wav_path"])
    wavs[row["clip_id"]] = wav

# Filter to violent She-Proves clips only
sp_violent = sp_clips[sp_clips["has_violence"] == True]

# Get per-backend split
sp_clips["backend"] = sp_clips["voice_families"].apply(
    lambda v: "google" if "Chirp" in v else "azure"
)
print(sp_clips.groupby("backend")["clip_id"].count())
# azure    10
# google    2
```

---

## Guidance for model training

!!! warning "This is a toy corpus — not for production training"
    12 She-Proves clips (10 Azure + 2 Google) are not enough for training a production model. Use this delivery to validate your data pipeline and schema parsing. Full-scale data follows.

**High-recall orientation:**

- **NEG clips are your hardest negatives.** They contain intense speech (raised voices, arguments, crying) with `has_violence: false`. Your recall model must not fire on them.
- **The pre-incident window** (first 60% of the clip) will look like NEU/low-intensity speech. Include it in your training windows — models that only see escalated segments will miss early warning signals.
- **Per-turn intensity** in the `.jsonl` events gives you fine-grained supervision beyond binary `has_violence`. Consider training an intensity regressor as an auxiliary objective.

**Backend diversity:**

The 2 Google Chirp HD clips expose your feature extractor to a different F0 baseline and spectral profile. At small scale, they're useful for checking that your features don't overfit to Azure voice characteristics.

**Speaker splits:**

All 12 clips share 2 unique speaker personas (4 if you count Azure+Google pairs separately). There are not enough speakers for a speaker-disjoint split in this delivery. Re-evaluate when the corpus scales to 100+ speakers.
