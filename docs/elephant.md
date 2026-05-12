# Elephant in the Room Guide

Elephant in the Room (הפיל שבחדר) is a Raspberry Pi–class device placed in clinic and welfare offices that alerts security when a social worker is under threat. **Optimisation target: high precision** — false alarms erode trust with the security team and the workers they protect.

This page is the *differential* between Elephant clips and the rest of the corpus. For shared concepts (schema, labels, audio format) follow the cross-links.

---

## Scene profile

| | |
|---|---|
| Project code | `elephant_in_the_room` (clip-id prefix `el_*`) |
| Tier | B — room IR + device profile + background noise applied |
| Duration | 1–4 min |
| Alert window | Final 40% of the clip — violence events concentrate here |
| Device | `pi_budget_mic` |
| Room types | `clinic_office`, `welfare_office`, `open_office` (only `clinic_office` in delivery-003) |

The alert-in-final-40% constraint mirrors real-world deployment: the device picks up normal consultation audio for most of the session before any threat emerges. The model must distinguish escalation from a baseline of routine professional interaction.

---

## What Tier B adds (and why)

Tier B clips run through three augmentation stages after preprocessing. This is what separates them from She-Proves (Tier A) clips.

| Stage | What it adds | Where to find it in metadata |
|-------|--------------|------------------------------|
| Room IR convolution | Reverb of a real-sounding room | `acoustic_scene.room_type`, `ir_source` |
| Device profile | Frequency response of a budget Pi microphone | `acoustic_scene.device` |
| Background noise injection | HVAC hum + occasional `ACOU_*` events | `acoustic_scene.background_events` |

After augmentation the clip is renormalised to the same peak target (–2.0 dBFS) as Tier A, so the two tiers are comparable on the loudness dimension.

!!! info "What `pyroomacoustics_ism` is"
    The image-source method (ISM) synthesises a room impulse response by simulating a virtual point source reflecting off the walls of a modelled room. [`pyroomacoustics`](https://pyroomacoustics.readthedocs.io/) is the Python library that implements it. The resulting IR, when convolved with clean speech, makes the speech sound like it was recorded in the modelled room — without needing a real recording.

---

## The `acoustic_scene` field

For Tier A clips this is all `null` / empty. For Elephant clips it's fully populated:

```json
"acoustic_scene": {
  "room_type":                "clinic_office",
  "device":                   "pi_budget_mic",
  "ir_source":                "pyroomacoustics_ism",
  "snr_db_actual":            11.2,
  "speaker_distance_meters":  1.2,
  "background_events": [
    {"type": "hvac_hum",  "onset":  0.000, "offset": 147.031, "level_db": -37.4},
    {"type": "ACOU_SLAM", "onset": 72.164, "offset":  72.476, "level_db":   9.9},
    {"type": "ACOU_FALL", "onset": 97.570, "offset":  98.473, "level_db":   9.6}
  ]
}
```

| Field | What it tells you |
|-------|-------------------|
| `room_type` | Modelled room (`clinic_office` / `welfare_office` / `open_office`) |
| `device` | Microphone profile applied (`pi_budget_mic`) |
| `ir_source` | How the room IR was generated (currently always `pyroomacoustics_ism`) |
| `snr_db_actual` | Measured speech-to-noise ratio in dB **after** mixing — your ground truth for SNR-stratified eval |
| `speaker_distance_meters` | Simulated distance from speaker to microphone |
| `background_events` | List of non-speech acoustic events: `hvac_hum` (constant low-level), `ACOU_SLAM` / `ACOU_FALL` (brief, high-level) |

!!! info "`ACOU_*` events are double-recorded"
    Each `ACOU_SLAM` / `ACOU_FALL` event lives in **both** `acoustic_scene.background_events` (with `level_db` mixing metadata) **and** the `.jsonl` strong-label file (as a regular `EventLabel` with `tier1_category: "ACOU"`). The two views are deliberate — the first carries audio-level provenance, the second is the supervision target. If you train an event detector, use the `.jsonl` view.

---

## Speaker pair

One pair in delivery-003. Roles match Israeli welfare/clinic demographics: BEN (client/service-user, male) + SW (social worker, female).

Speaker directory: `data/he/ben_m_40-55_003/`

| Role | speaker_id | TTS voice |
|------|-----------|-----------|
| BEN | `BEN_M_40-55_003` | `he-IL-AvriNeural` |
| SW  | `SW_F_30-45_001`  | `he-IL-HilaNeural` |

Both speakers use the Azure backend. See [Glossary — Speaker roles](glossary.md#speaker-roles) if `BEN` and `SW` are new abbreviations.

---

## Clips in delivery-003

**8 clips · ~17 min · 4 violent (SV + IT), 4 non-violent (NEG + NEU) · all `room_type: clinic_office`, all `device: pi_budget_mic`, SNR ~11 dB**

??? abstract "Full clip listing"
    All in `data/he/ben_m_40-55_003/`:

    | Clip ID | Typology | violent | Duration |
    |---------|----------|:---:|---------:|
    | `el_sv_b_0001_00`  | SV  | ✓ | 2m 27.0s |
    | `el_sv_b_0002_00`  | SV  | ✓ | 2m 18.5s |
    | `el_it_b_0001_00`  | IT  | ✓ | 2m 30.0s |
    | `el_it_b_0002_00`  | IT  | ✓ | 2m 31.6s |
    | `el_neg_b_0001_00` | NEG | — | 1m 53.8s |
    | `el_neg_b_0002_00` | NEG | — | 2m 54.6s |
    | `el_neu_b_0001_00` | NEU | — | 1m 56.9s |
    | `el_neu_b_0002_00` | NEU | — | 1m 19.7s |

---

## Loading and inspecting an Elephant clip

```python
import pandas as pd, soundfile as sf, json
from pathlib import Path

root = Path(".")
df = pd.read_csv("data/he/manifest.csv")
el = df[df["project"] == "elephant_in_the_room"]    # 8 rows

# Pick one clip
row = el.iloc[0]
wav, sr = sf.read(root / row.wav_path)
meta = json.loads((root / row.wav_path).with_suffix(".json").read_text())

# Acoustic scene
scene = meta["acoustic_scene"]
print(f"{scene['room_type']}  {scene['device']}  SNR {scene['snr_db_actual']} dB  "
      f"dist {scene['speaker_distance_meters']} m")

# Background acoustic events
for evt in scene["background_events"]:
    print(f"  {evt['type']:10s}  {evt['onset']:6.1f}s – {evt['offset']:6.1f}s  @ {evt['level_db']:+5.1f} dB")

# Alert window (final 40%) — for sliding-window evaluation
duration = meta["duration_seconds"]
alert_start = 0.60 * duration

events = [json.loads(l) for l in (root / row.strong_labels_path).read_text().splitlines() if l.strip()]
alert_events = [e for e in events if e["onset"] >= alert_start]
print(f"alert window: {alert_start:.1f}s – {duration:.1f}s   "
      f"{len(alert_events)} events fire in window  "
      f"(of {len(events)} total)")
```

---

## Training-time notes (specific to this project)

- **NEG clips are essential for precision.** `el_neg_b_*` is intense speech in a clinic room with background noise but no violence. If your detector fires on these, security stops trusting it. Train hard against these.
- **The alert-in-final-40% structure is exploitable.** Consider a sliding-window detector that biases toward the back half of each clip — or use the window structure as a positional feature. Don't reward early firing.
- **SNR ~11 dB is challenging.** Verify your features (MFCCs, log-mel, etc.) are robust here before comparing with She-Proves Tier A results. SNR is recorded per clip (`acoustic_scene.snr_db_actual`) — use it for SNR-stratified eval.
- **`ACOU_*` events double as strong labels.** You can train an event detector on `ACOU_SLAM` / `ACOU_FALL` separately from the speech-violence detector and ensemble them.
- **What delivery-003 *doesn't* cover:** only `clinic_office`, only one speaker pair (BEN+SW), only Azure backend, SNR essentially constant at ~11 dB. Plan for room diversity, SNR stratification, and speaker-disjoint splits when scaling.

!!! warning "Still a small test batch"
    8 clips, 1 room type, 1 speaker pair, 1 SNR is enough to wire up data loaders and acoustic-scene parsing. It is not enough to train a production model. Build the plumbing; wait for the real batch.
