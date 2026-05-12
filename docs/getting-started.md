# Getting Started

This guide walks through loading and using clips from the corpus in Python. All paths are relative to the repository root.

## Prerequisites

```bash
pip install soundfile numpy pandas pydantic
```

??? note "Optional: full SynthBanshee schema"
    If you want strict Pydantic validation against the full `ClipMetadata` schema:
    ```bash
    git clone https://github.com/DataHackIL/SynthBanshee
    cd SynthBanshee && pip install -e .
    ```
    This gives you `from synthbanshee.labels.schema import ClipMetadata` and `validate_clip()`.
    For most DS workflows, plain `json.loads()` is sufficient.

## Clone the corpus

```bash
git clone https://github.com/DataHackIL/avdp-synth-corpus.git
cd avdp-synth-corpus
```

The repository contains the audio files directly (no LFS). Total size is moderate — `data/he/` is roughly a few hundred MB for delivery-003.

---

## Load a single clip

```python
import json
from pathlib import Path
import soundfile as sf
import numpy as np

root = Path(".")  # run from repo root

clip_id = "sp_sv_a_0001_00"
speaker_dir = root / "data/he/agg_m_30-45_001"

# --- Audio ---
wav, sr = sf.read(speaker_dir / f"{clip_id}.wav")
# wav: float64 array, shape (N,). sr: always 16000.

print(f"Duration: {len(wav)/sr:.1f}s  Sample rate: {sr}  Peak: {np.abs(wav).max():.4f}")
# Duration: 110.5s  Sample rate: 16000  Peak: 0.7943

# --- Weak labels (ClipMetadata) ---
meta = json.loads((speaker_dir / f"{clip_id}.json").read_text())
wl = meta["weak_label"]
print(f"Typology: {meta['violence_typology']}  has_violence: {wl['has_violence']}  "
      f"max_intensity: {wl['max_intensity']}")
# Typology: SV  has_violence: True  max_intensity: 5

# --- Transcript ---
transcript = (speaker_dir / f"{clip_id}.txt").read_text(encoding="utf-8")
print(transcript[:200])  # Hebrew turns with timestamps
```

??? info "Why is the peak ~0.79 (–2.0 dBFS) not 1.0?"
    All clips are peak-normalized to a **–2.0 dBFS target** (not –1.0 dBFS = 1.0 linear).
    This gives 2 dB of headroom above the safety limiter ceiling (–1.0 dBFS).
    `preprocessing_applied.normalized_dbfs` in the JSON records the measured peak.
    See [Audio Format](audio-format.md) for the full normalization pipeline.

---

## Load strong-label events

```python
import jsonlines  # pip install jsonlines

events = []
with jsonlines.open(speaker_dir / f"{clip_id}.jsonl") as reader:
    for evt in reader:
        events.append(evt)

# Or without jsonlines:
events = [
    json.loads(line)
    for line in (speaker_dir / f"{clip_id}.jsonl").read_text().splitlines()
    if line.strip()
]

for evt in events[:3]:
    print(f"[{evt['onset']:.1f}s – {evt['offset']:.1f}s] "
          f"{evt['tier1_category']}/{evt['tier2_subtype']}  I{evt['intensity']}")
# [0.8s – 10.1s] VERB/VERB_SHOUT  I2
# [10.5s – 18.7s] VERB/VERB_SHOUT  I2
# [18.3s – 29.7s] VERB/VERB_THREAT  I3
```

??? info "What are tier1_category and tier2_subtype?"
    Strong labels follow a three-level taxonomy:

    **Typology** (clip-level): `SV` · `IT` · `NEG` · `NEU`

    **Tier 1 category** (event-level): `VERB` · `DIST` · `PHYS` · `EMOT` · `ACOU` · `NONE`

    **Tier 2 subtype** (event-level): e.g. `VERB_SHOUT`, `VERB_THREAT`, `DIST_SCREAM`, `PHYS_HARD`, `ACOU_SLAM`

    See [Label Taxonomy](taxonomy.md) for the full table and has_violence derivation rule.

---

## Work with the manifest

`data/he/manifest.csv` is a flat summary of all clips. It's the fastest entry point for filtering and dataset construction.

```python
import pandas as pd

df = pd.read_csv("data/he/manifest.csv")
print(df.columns.tolist())
# ['clip_id', 'project', 'violence_typology', 'tier', 'duration_seconds',
#  'speaker_ids', 'voice_families', 'has_violence', 'max_intensity',
#  'quality_flags', 'split', 'wav_path', 'strong_labels_path']

# Filter by project
she_proves_clips = df[df["project"] == "she_proves"]

# Filter by typology
sv_clips = df[df["violence_typology"] == "SV"]

# High-intensity violent clips only
high_intensity = df[(df["has_violence"]) & (df["max_intensity"] >= 4)]

# Load audio for a manifest row
row = df.iloc[0]
wav, sr = sf.read(row["wav_path"])  # paths are repo-relative POSIX strings
```

!!! warning "`speaker_ids` and `voice_families` are pipe-delimited"
    These columns contain multiple values joined by `|`:
    ```python
    speakers = row["speaker_ids"].split("|")
    # ['AGG_M_30-45_001', 'VIC_F_25-40_002']
    ```

!!! note "All clips are `split: train` in delivery-003"
    The corpus has only 4 unique speaker personas across 20 clips — speaker-disjoint splits are not feasible at this scale. When the corpus scales, speaker-disjoint train/val/test splits will be assigned by SynthBanshee. Until then, treat this as an unpartitioned pool.

---

## Find a clip's speaker directory

Clip IDs follow the pattern `{project_prefix}_{typology}_{tier}_{scene_num}_{take}`. The on-disk directory is the **lowercase** form of the first speaker ID listed in `speakers[]`:

```python
def clip_dir(root: Path, clip_id: str, meta: dict) -> Path:
    first_speaker = meta["speakers"][0]["speaker_id"]
    return root / "data" / meta["language"] / first_speaker.lower()
```

| clip_id | speaker_dir |
|---------|-------------|
| `sp_sv_a_0001_00` | `data/he/agg_m_30-45_001/` |
| `sp_sv_a_0003_00` | `data/he/agg_m_30-45_002/` |
| `el_sv_b_0001_00` | `data/he/ben_m_40-55_003/` |

Or use `manifest.csv` directly — `wav_path` already contains the full repo-relative path.

---

## Validate a clip

If you have SynthBanshee installed:

```bash
synthbanshee validate data/he/agg_m_30-45_001/sp_sv_a_0001_00.wav
```

This checks: all four files present, WAV format (16 kHz mono), peak ≤ –1.0 dBFS, duration ≥ 3 s, JSON parses as `ClipMetadata`.

To run QA over the entire language directory:

```bash
synthbanshee qa-report data/he/
synthbanshee qa-report data/he/ --run-summary   # adds corpus-level aggregates
```
