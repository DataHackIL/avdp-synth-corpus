# She-Proves Guide

She-Proves is a smartphone app that passively monitors audio for domestic-violence incidents and preserves evidence for legal use. **Optimisation target: high recall** — better to flag for review than to miss.

This page is the *differential* between She-Proves clips and the rest of the corpus. For shared concepts (schema, labels, audio format) follow the cross-links.

---

## Scene profile

| | |
|---|---|
| Project code | `she_proves` (clip-id prefix `sp_*`) |
| Tier | A — clean audio, no room/device augmentation |
| Duration | 3–6 min |
| Pre-incident window | ≥ 60% of clip is normal speech before the first violence event |
| Device | `phone_in_pocket`, `phone_on_table`, `phone_in_hand` (planned; not active in delivery-003) |
| Room | Apartment (living room, bedroom, kitchen) — planned; not active in delivery-003 |

The long pre-incident window is intentional. In deployment the app is always listening; incidents are rare. A model trained only on escalation segments will miss the gradual-buildup signal that precedes most domestic-violence events.

!!! note "What 'Tier A' means here"
    Tier A audio is the direct TTS-mixer output after preprocessing — peak-normalised, silence-padded, 16 kHz mono PCM. No room IR, no microphone profile, no background noise. `acoustic_scene.room_type`, `device`, `ir_source`, and `snr_db_actual` are all `null` for every Tier A clip.

    Delivery-003 has no Tier-A device augmentation yet (the `phone_in_pocket` etc. profiles exist in the pipeline but aren't applied at this stage). When that's added in a future delivery, the `acoustic_scene` block will start carrying `device` while keeping `room_type` null.

---

## Speaker pairs

Two pairs in delivery-003, one per TTS backend. Both pairs play the **AGG (aggressor, male) + VIC (victim, female)** roles.

=== "Azure pair (10 clips)"
    Speaker directory: `data/he/agg_m_30-45_001/`

    | Role | speaker_id | TTS voice |
    |------|-----------|-----------|
    | AGG  | `AGG_M_30-45_001` | `he-IL-AvriNeural` |
    | VIC  | `VIC_F_25-40_002` | `he-IL-HilaNeural` |

=== "Google Chirp HD pair (2 clips)"
    Speaker directory: `data/he/agg_m_30-45_002/`

    | Role | speaker_id | TTS voice |
    |------|-----------|-----------|
    | AGG  | `AGG_M_30-45_002` | `he-IL-Chirp3-HD-Achird`   |
    | VIC  | `VIC_F_25-40_003` | `he-IL-Chirp3-HD-Achernar` |

    The Google pair was added in delivery-003 specifically to introduce backend diversity. Both clips carry a `vic_f0_high` flag — see [Audio Format](audio-format.md#vic_f0_high-on-the-2-google-clips).

[Gotcha #2: don't hardcode `agg_m_30-45_001/`](gotchas.md#2-dont-hardcode-speaker-directory-paths) — three speaker directories exist now, including one for Elephant. Filter on `manifest.csv["project"] == "she_proves"` or on `meta["project"]`.

---

## Clips in delivery-003

**12 clips · ~20 min · 6 violent (`SV` + `IT`), 6 non-violent (`NEG` + `NEU`)**

??? abstract "Full clip listing"
    Azure pair, `data/he/agg_m_30-45_001/` — 10 clips:

    | Clip ID | Typology | violent | Duration |
    |---------|----------|:---:|---------:|
    | `sp_sv_a_0001_00`  | SV  | ✓ | 1m 50.5s |
    | `sp_sv_a_0002_00`  | SV  | ✓ | 1m 32.1s |
    | `sp_it_a_0001_00`  | IT  | ✓ | 2m 23.8s |
    | `sp_it_a_0002_00`  | IT  | ✓ | 2m 19.7s |
    | `sp_neg_a_0001_00` | NEG | — | 1m 58.8s |
    | `sp_neg_a_0002_00` | NEG | — | 1m 47.8s |
    | `sp_neg_a_0003_00` | NEG | — | 2m 26.3s |
    | `sp_neu_a_0001_00` | NEU | — | 1m 59.2s |
    | `sp_neu_a_0002_00` | NEU | — | 2m 09.0s |
    | `sp_neu_a_0003_00` | NEU | — | 1m 45.1s |

    Google Chirp HD pair, `data/he/agg_m_30-45_002/` — 2 clips:

    | Clip ID | Typology | violent | Duration | Flags |
    |---------|----------|:---:|---------:|-------|
    | `sp_sv_a_0003_00` | SV | ✓ | 1m 42.8s | `vic_f0_high` |
    | `sp_it_a_0003_00` | IT | ✓ | 1m 53.9s | `vic_f0_high` |

The waveform on the [home page](index.md#see-it-first) is `sp_sv_a_0001_00` — a worked example of an SV escalation arc in this project's data.

---

## Loading just the She-Proves clips

```python
import pandas as pd, soundfile as sf, json
from pathlib import Path

root = Path(".")
df = pd.read_csv("data/he/manifest.csv")
sp = df[df["project"] == "she_proves"]                   # 12 rows

# Tag backend per row (Google clips have "Chirp" in voice_families)
sp = sp.assign(backend=sp["voice_families"].str.contains("Chirp").map({True: "google", False: "azure"}))
print(sp.groupby("backend")["clip_id"].count())
# azure     10
# google     2

# Load audio for each row
audio = {row.clip_id: sf.read(root / row.wav_path) for row in sp.itertuples()}
```

---

## Training-time notes (specific to this project)

- **NEG clips are your hardest negatives.** `sp_neg_a_*` clips have raised voices, distress, arguments — and `has_violence: false`. Recall metrics that fire on these will tank precision. See [Gotcha #1](gotchas.md#1-dont-derive-has_violence-from-typology) and [Gotcha #5](gotchas.md#5-neg-is-not-violent-at-low-intensity).
- **Use the pre-incident window.** The first 60% of each violent clip looks like NEU-grade speech. Train across the full clip, not only on escalation segments — early-warning signal lives in the buildup.
- **Per-turn intensity is a useful auxiliary objective.** `EventLabel.intensity` gives turn-level supervision beyond binary `has_violence`. An intensity regressor trained alongside the classifier often boosts the latter.
- **Only 2 voice families per gender in this delivery** (`low_voice_diversity_*` is flagged at the corpus level). Expect your acoustic features to over-fit to AvriNeural and HilaNeural — track per-voice eval separately when the corpus grows.
- **No device/room augmentation yet on She-Proves clips.** When the `phone_in_pocket` profile activates in a future delivery, your model will see substantially more high-frequency roll-off and handling noise than what's in delivery-003.

!!! warning "Still a small test batch"
    12 clips and 4 voices is enough to wire up your data loaders, label parsers, and evaluation harness. It is not enough to train a production model. Build the plumbing; wait for the real batch.
