# Common mistakes

Read this once before you write code against the corpus. Two minutes here saves a debugging session later.

---

## 1. Don't derive `has_violence` from typology

This will misclassify every `NEG` clip:

```python
# WRONG — NEG clips will look violent because of their max_intensity
has_violence = typology in ("SV", "IT")

# CORRECT — uses the event-level ground truth
has_violence = any(e["tier1_category"] != "NONE" for e in events)
```

`has_violence` in `weak_label` is **derived from strong-label events**, not from typology. NEG clips can have `max_intensity = 3` (raised voices, distress) and still be `has_violence: false` because every one of their events lands `tier1_category: "NONE"` by design. That's the whole point of NEG: hard negatives that sound intense but aren't violent.

---

## 2. Don't hardcode speaker directory paths

There's already more than one. Delivery-003 has three speaker directories under `data/he/`:

```
data/he/agg_m_30-45_001/   # She-Proves, Azure pair
data/he/agg_m_30-45_002/   # She-Proves, Google Chirp HD pair  (new in delivery-003)
data/he/ben_m_40-55_003/   # Elephant in the Room, Azure pair  (new in delivery-003)
```

Code that hardcodes `data/he/agg_m_30-45_001/` will miss two-thirds of the clips. Use `manifest.csv` (the `wav_path` column is repo-relative POSIX), or derive the directory from the first entry in `speakers[]`:

```python
speaker_dir = root / "data" / meta["language"] / meta["speakers"][0]["speaker_id"].lower()
```

---

## 3. Audio peak is ~0.79, not 1.0

Clips are normalized to a **–2.0 dBFS peak target** (not –1.0 dBFS = linear 1.0). Loading a clip and expecting full-range float values will surprise you:

```python
wav, sr = sf.read("data/he/agg_m_30-45_001/sp_sv_a_0001_00.wav")
print(np.abs(wav).max())  # ~0.7943, not 1.0
```

The –2 dBFS target leaves 2 dB of headroom above the safety limiter at –1.0 dBFS. The configured target is recorded in `generation_metadata.loudness_target_peak_dbfs`; the measured peak is recorded in `preprocessing_applied.normalized_dbfs`.

---

## 4. UPPERCASE in JSON, lowercase on disk

The same speaker has two surface forms:

| Surface | Form | Example |
|---------|------|---------|
| JSON field (`speaker_id`, `speakers[].speaker_id`) | **UPPERCASE** | `AGG_M_30-45_001` |
| Filesystem directory | **lowercase** | `agg_m_30-45_001/` |
| `clip_id` (everywhere) | **lowercase** | `sp_sv_a_0001_00` |

If you build a dict keyed on speaker IDs from JSON and then try to look up paths with the same string, you'll get a `FileNotFoundError`. Always `.lower()` when converting from JSON to a path.

---

## 5. NEG is not "violent at low intensity"

The four violence typologies are **not** an ordered scale.

| | |
|---|---|
| `SV` | Severe Violence — physical attacks, life-threatening |
| `IT` | Intimate Terrorism — sustained coercive control, repeated abuse |
| `NEG` | **Negative confusor** — sounds intense, no violence (hard negative) |
| `NEU` | Neutral — mundane conversation |

A NEG clip is **not** "a milder SV." It is acoustic distress that a naive model would mistake for violence. Treating NEG as a positive class will tank your precision.

---

## 6. All clips are `split: train` in delivery-003

The `split` column exists in `manifest.csv`, but there are only 4 unique speaker personas across all 20 clips. Speaker-disjoint train/val/test partitioning isn't feasible at this scale — every clip is therefore assigned `split: train`. **Don't trust the `split` column as a usable partition.** Treat the whole corpus as an unpartitioned pool for now. SynthBanshee will assign meaningful splits once the speaker pool grows.

---

## 7. `quality_flags` doesn't mean "broken"

About 15 of 20 clips in delivery-003 carry at least one `quality_flags` entry — usually `emotion_downgrade` (the TTS produced slightly less intense prosody than the SSML asked for at high-intensity turns). These clips are still validated and spec-compliant; the flag is a soft hint, not a failure. Don't filter them out reflexively.

The hard line is `synthbanshee validate` — a clip either passes or doesn't. If it's in the corpus, it passed.

---

## 8. The 2 Google clips have a `vic_f0_high` flag — that's expected

`sp_sv_a_0003_00` and `sp_it_a_0003_00` use the Google Chirp 3 HD female voice (`he-IL-Chirp3-HD-Achernar`), whose fundamental-frequency baseline runs higher than the Azure reference voice the QA thresholds were calibrated against. The flag is fired correctly; the audio is fine. **Don't exclude these clips on the basis of this flag** — your model needs the backend diversity. If you compute F0-derived features, calibrate per backend.

---

## 9. Timestamps already account for silence padding

Every clip has ≥0.5 s of silence at head and tail. **Onset/offset timestamps in `.txt` and `.jsonl` are already shifted** to refer to positions in the final processed WAV. You don't need to add the pad — read the timestamp, slice the WAV, done.

---

## 10. The `.json` and `.jsonl` files aren't the same thing

| File | Contains | When to load |
|------|----------|--------------|
| `{clip_id}.json` | `ClipMetadata` — one object per clip: weak labels, speakers, provenance, acoustic scene | Always |
| `{clip_id}.jsonl` | `EventLabel` records — one JSON object per **line**, one per labelled event in the clip | When you need per-event strong labels (onset/offset/category) |

If you `json.loads()` the `.jsonl` you'll get an error. Read line by line.

---

## Quick verification

Use these snippets to confirm a fresh clone is intact:

```bash
find data/he -name "*.wav" | wc -l     # expect 20
wc -l data/he/manifest.csv              # expect 21 (header + 20 rows)
```

```python
import pandas as pd
df = pd.read_csv("data/he/manifest.csv")
assert len(df) == 20
assert set(df["tier"]) == {"A", "B"}
assert set(df["violence_typology"]) == {"SV", "IT", "NEG", "NEU"}
assert df["has_violence"].sum() == 10   # 5 SV + 5 IT
```
