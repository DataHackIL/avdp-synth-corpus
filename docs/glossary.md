# Glossary

Abbreviations and jargon that show up across the corpus and on this site, in one place.

---

## Speaker roles

The role of each speaker is encoded in the speaker_id prefix and in `speakers[].role`.

| Code | Stands for | Used in |
|------|-----------|---------|
| `AGG` | **Aggressor** — the perpetrator in a domestic-violence scene | She-Proves clips (`AGG_M_30-45_*`) |
| `VIC` | **Victim** — the target of violence in a domestic-violence scene | She-Proves clips (`VIC_F_25-40_*`) |
| `BEN` | **Beneficiary / client** — a service-user in a welfare or clinic setting (the threatening party in Elephant scenes) | Elephant clips (`BEN_M_40-55_*`) |
| `SW` | **Social Worker** — the threatened professional in Elephant scenes | Elephant clips (`SW_F_30-45_*`) |

The role determines the prosody profile, scene position, and which `tier1_category` events the speaker can produce.

---

## Project codes

| Code | Project | Clip ID prefix |
|------|---------|----------------|
| `she_proves` | She-Proves smartphone app | `sp_*` |
| `elephant_in_the_room` | Elephant in the Room (clinic/welfare device) | `el_*` |

---

## Violence typology

The clip-level `violence_typology` field — not an ordered scale. See [Label Taxonomy](taxonomy.md) for details.

| Code | Stands for |
|------|------------|
| `SV` | Severe Violence |
| `IT` | Intimate Terrorism |
| `NEG` | Negative confusor (sounds intense, no violence) |
| `NEU` | Neutral |

---

## Tier 1 event category

The event-level `tier1_category` field on each `EventLabel`.

| Code | Stands for |
|------|------------|
| `VERB` | Verbal violence (shouting, threats, insults) |
| `DIST` | Distress vocalisations (screaming, crying under duress) |
| `PHYS` | Physical violence cues (impact sounds, struggle) |
| `EMOT` | Emotional manipulation (gaslighting, guilt-tripping) |
| `ACOU` | Acoustic non-vocal events (slams, falls) |
| `NONE` | Ambient / neutral / no violence cue |

---

## Tier codes

| Code | Meaning |
|------|---------|
| `A` | Clean audio — no room IR, no device profile, no background noise |
| `B` | Room IR + device profile + background noise injection |

---

## Audio jargon

| Term | Meaning |
|------|---------|
| **F0** | Fundamental frequency — the lowest frequency of a periodic signal; for voice, the pitch. Reported per speaker in some QA outputs. |
| **dBFS** | Decibels relative to full scale — 0 dBFS is the maximum amplitude representable by the format; –2 dBFS is ~80% of full amplitude. |
| **Peak normalization** | Applying a single gain to the whole signal so its absolute maximum matches a target level. |
| **RMS** | Root-mean-square — a measure of average signal energy. SynthBanshee uses per-turn RMS gain to enforce the loudness gradient between calm and escalated turns. |
| **SNR** | Signal-to-noise ratio — speech level minus background-noise level, in dB. Recorded in `acoustic_scene.snr_db_actual` for Tier B clips. |
| **IR** | Impulse response — a recording of how a room (or microphone, or speaker) responds to an idealised pulse. Convolving clean speech with a room IR makes it sound like it was recorded in that room. |
| **ISM** | Image-source method — an algorithm for synthetically generating room IRs by reflecting virtual sound sources off room walls. Implemented by `pyroomacoustics`. |
| **SSML** | Speech Synthesis Markup Language — an XML dialect that controls TTS output (pitch, rate, emphasis, breaks, voice). Azure and Google both accept SSML. |
| **TTS** | Text-to-speech — the generation of audio from a text prompt. |
| **Prosody** | The patterns of stress, intonation, pitch, and rate that make speech expressive (vs. flat). |
| **Prosody cap** | A safety clamp applied by SynthBanshee to LLM-suggested prosody values to prevent unnatural extremes (pitch ≤ +2 st, rate ∈ [0.85, 1.20]). |
| **Whisper** | OpenAI's open-weight ASR model, used internally as a sanity check that synthesised audio is still transcribable. |

---

## Pipeline / corpus jargon

| Term | Meaning |
|------|---------|
| **Dirty file** | The pre-preprocessing WAV (raw TTS-mixer output, before normalization and padding). Retained under `assets/speech/dirty/{clip_id}_dirty.wav`. |
| **Generation metadata** | The `generation_metadata` field — pipeline provenance: which TTS backend was used, which voice family, what mix mode, etc. |
| **Manifest** | The flat CSV summary at `data/he/manifest.csv` — one row per clip, columns for filtering. |
| **Strong labels** | Event-level labels in `.jsonl` files — one `EventLabel` object per labelled event, with onset/offset/category. |
| **Weak labels** | Clip-level summary labels in `.json` — `has_violence`, `max_intensity`, `violence_typology`, `violence_categories`. |
| **Quality flag** | A soft warning in `quality_flags` (e.g. `emotion_downgrade`). Doesn't fail validation; flags audio worth a second look. |
| **Delivery** | A merged data batch under `deliveries/{slug}/`. Each delivery records its SynthBanshee commit, metadata, and per-batch QA notes. |

---

## Hebrew TTS voice IDs

The four voices used in delivery-003:

| Voice ID | Gender | Backend |
|----------|:---:|---------|
| `he-IL-AvriNeural` | M | Azure |
| `he-IL-HilaNeural` | F | Azure |
| `he-IL-Chirp3-HD-Achird` | M | Google Chirp 3 HD |
| `he-IL-Chirp3-HD-Achernar` | F | Google Chirp 3 HD |
