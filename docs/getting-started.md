# Start here

Your first 10 minutes with the corpus. By the end you'll have cloned it, verified the clone, loaded one clip with its labels, and seen what's in a transcript.

---

## 1. Clone

```bash
git clone https://github.com/DataHackIL/avdp-synth-corpus.git
cd avdp-synth-corpus
```

No Git LFS. Total size is a few hundred megabytes for delivery-003 — the audio lives in `data/he/`, the SSML caches live in `assets/`.

---

## 2. Verify the clone

```bash
find data/he -name "*.wav" | wc -l    # expect 20
wc -l data/he/manifest.csv            # expect 21 (header + 20 rows)
```

If those numbers don't match, the clone is incomplete — `git lfs pull` is not the answer (we don't use LFS). Re-clone.

---

## 3. Install the minimal Python deps

```bash
pip install soundfile numpy pandas
```

That's enough for everything on this page. `pydantic` is only needed if you want strict schema validation; `jsonlines` only if you prefer it to the one-liner that reads `.jsonl` directly.

??? note "When you'd want the full SynthBanshee install"
    If you want `from synthbanshee.labels.schema import ClipMetadata` for strict Pydantic validation, or `synthbanshee qa-report` to re-run QA over the data directory:
    ```bash
    git clone https://github.com/DataHackIL/SynthBanshee
    cd SynthBanshee && pip install -e .
    ```
    For consuming the corpus, `json.loads()` is fine and is what the examples below use.

---

## 4. Load one clip end-to-end

The path on disk is **lowercase** even though the speaker ID in JSON is **UPPERCASE** — that's a [Gotcha #4](gotchas.md#4-uppercase-in-json-lowercase-on-disk).

```python
import json
from pathlib import Path
import soundfile as sf

root = Path(".")                          # repo root
clip_dir = root / "data/he/agg_m_30-45_001"
clip_id = "sp_sv_a_0001_00"

# Audio
wav, sr = sf.read(clip_dir / f"{clip_id}.wav")
assert sr == 16000 and wav.ndim == 1      # always 16 kHz mono

print(f"duration={len(wav)/sr:.1f}s  peak={abs(wav).max():.3f}")
# duration=110.5s  peak=0.794
```

!!! info "Why is the peak 0.794 and not 1.0?"
    Clips are normalized to a **–2.0 dBFS peak target**, which is roughly 0.79 linear amplitude. Use `generation_metadata.loudness_target_peak_dbfs` to read the configured target and `preprocessing_applied.normalized_dbfs` to read the measured output peak. Full detail: [Audio Format](audio-format.md).

Clip-level labels (weak labels):

```python
meta = json.loads((clip_dir / f"{clip_id}.json").read_text())
wl = meta["weak_label"]
print(f"typology={meta['violence_typology']}  has_violence={wl['has_violence']}  "
      f"intensity_max={wl['max_intensity']}  categories={wl['violence_categories']}")
# typology=SV  has_violence=True  intensity_max=5  categories=['DIST', 'PHYS', 'VERB']
```

Event-level labels (strong labels):

```python
events = [
    json.loads(line)
    for line in (clip_dir / f"{clip_id}.jsonl").read_text().splitlines()
    if line.strip()
]

for evt in events[:3]:
    print(f"[{evt['onset']:5.1f}s – {evt['offset']:5.1f}s]  "
          f"{evt['speaker_role']}  {evt['tier1_category']}/{evt['tier2_subtype']}  "
          f"I{evt['intensity']}")
# [  0.8s –  10.1s]  AGG  VERB/VERB_SHOUT   I2
# [ 10.5s –  18.7s]  VIC  VERB/VERB_SHOUT   I2
# [ 18.3s –  29.7s]  AGG  VERB/VERB_THREAT  I3
```

The full 14-event escalation arc for this clip is the one visualised on the [home page](index.md#see-it-first) — verbal → distress → physical → settle.

---

## 5. Read a transcript

`.txt` files are turn-major with a small header block per turn. They use UTF-8 Hebrew and are intended both for human reading and as ASR reference.

```
[CLIP_ID: sp_sv_a_0001_00]
[SPEAKER: AGG_M_30-45_001 | ROLE: AGG | ONSET: 0.76 | OFFSET: 10.07]
מה זה הארוחה הזאת? שאלתי אותך דבר אחד פשוט, לעשות ארוחת ערב נורמלית.
[ACTION: VERB_SHOUT | INTENSITY: 2]
[SPEAKER: VIC_F_25-40_002 | ROLE: VIC | ONSET: 10.49 | OFFSET: 18.74]
עבדתי עד שש היום. עשיתי מה שהספקתי...
```

!!! note "Hebrew is right-to-left; some terminals mis-render it"
    macOS Terminal.app handles it correctly; older Windows consoles don't. If transcripts look reversed or garbled, view the `.txt` in an editor (VS Code, BBEdit) rather than `cat`.

Timestamps in the header are already relative to the **final processed WAV** — they include the 0.5 s silence pad at the head. No shift needed.

---

## 6. Work from the manifest, not from hardcoded paths

`data/he/manifest.csv` is one row per clip. It's the fastest entry point for filtering and the safest way to find files (because [hardcoded speaker directories will miss two-thirds of the clips](gotchas.md#2-dont-hardcode-speaker-directory-paths)).

```python
import pandas as pd, soundfile as sf

df = pd.read_csv("data/he/manifest.csv")
df.columns.tolist()
# ['clip_id', 'project', 'violence_typology', 'tier', 'duration_seconds',
#  'speaker_ids', 'voice_families', 'has_violence', 'max_intensity',
#  'quality_flags', 'split', 'wav_path', 'strong_labels_path']

# Filter
violent  = df[df["has_violence"]]                                 # 10 clips
elephant = df[df["project"] == "elephant_in_the_room"]            # 8 clips
sv_high  = df[(df["violence_typology"] == "SV") & (df["max_intensity"] >= 4)]

# Load audio for any manifest row — wav_path is already repo-relative POSIX
row = df.iloc[0]
wav, sr = sf.read(row["wav_path"])
speakers = row["speaker_ids"].split("|")        # pipe-delimited!
voices   = row["voice_families"].split("|")     # same order as speaker_ids
```

!!! warning "`speaker_ids` and `voice_families` are pipe-delimited strings"
    They are not CSV-nested lists. Split on `|`.

---

## 7. Where to go next

| You're about to… | Read |
|------------------|------|
| Write data-loading code | [Common mistakes](gotchas.md) (2 min read; saves debugging) |
| Look up a field in `.json` | [Schema Reference](schema.md) |
| Understand `has_violence` semantics | [Label Taxonomy](taxonomy.md) |
| Look up a term (F0, SSML, IR, BEN…) | [Glossary](glossary.md) |
| Work specifically with phone-app data | [She-Proves guide](she-proves.md) |
| Work specifically with Tier B / room-augmented audio | [Elephant in the Room guide](elephant.md) |
| Verify a clip is spec-compliant | `synthbanshee validate <path>` (requires SynthBanshee installed) |
