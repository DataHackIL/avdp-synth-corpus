# Label Taxonomy

Labels follow a three-level hierarchy. The **source of truth** is `taxonomy.yaml` in the [SynthBanshee](https://github.com/DataHackIL/SynthBanshee) repo. Never derive labels from field names alone — always read from the actual data.

---

## Violence typologies (clip-level)

The `violence_typology` field classifies the overall scenario of the clip.

| Typology | Full name | Description |
|----------|-----------|-------------|
| `SV` | Severe Violence | Physical violence, life-threatening escalation |
| `IT` | Intimate Terrorism | Systematic coercive control, repeated verbal/emotional abuse |
| `NEG` | Negative / Confusor | Acoustically intense but non-violent — anger, argument, distress, crying |
| `NEU` | Neutral | Calm or mundane conversation with no violence markers |

??? info "Why NEG is not the same as non-violent IT/SV"
    NEG clips are designed as **hard negatives** — they sound intense and may have raised voices, crying, or confrontational tone, but no actual violence occurs. Their purpose is to train models to distinguish acoustic distress from violence.

    Models that rely only on loudness or emotional tone will misclassify NEG clips. This is by design.

---

## `has_violence` — the correct derivation

`has_violence` is a **derived convenience field** computed from the strong-label events, not from typology:

```python
has_violence = any(e["tier1_category"] != "NONE" for e in events)
```

This means:

- `NEG` clips are **always** `has_violence: false`, regardless of `max_intensity` — by definition, every event in a NEG clip lands `tier1_category: "NONE"`.
- A `NEU` clip with even one stray non-NONE event would be `has_violence: true` (shouldn't happen in a well-labelled corpus, but the rule is defensive).

!!! danger "Do not re-derive `has_violence` from typology + intensity"
    ```python
    # WRONG — will misclassify every NEG clip
    has_violence = typology in ("SV", "IT")

    # CORRECT
    has_violence = any(e["tier1_category"] != "NONE" for e in events)
    ```
    The taxonomy columns are the ground truth. `has_violence` exists only for fast filtering and baseline modelling — never use it as the sole training label.

---

## Tier 1 categories (event-level)

Each `EventLabel` in the `.jsonl` file has a `tier1_category`:

| Category | Description | Example contexts |
|----------|-------------|-----------------|
| `VERB` | Verbal violence — threats, shouting, demeaning language | Arguments, intimidation |
| `DIST` | Distress vocalisations — screaming, crying under duress | Peak escalation turns |
| `PHYS` | Physical violence cues — impact sounds, struggle | Severe violence scenes |
| `EMOT` | Emotional manipulation — guilt-tripping, gaslighting | IT/coercive control |
| `ACOU` | Acoustic events — object impacts, slams, falls | Background events in Tier B |
| `NONE` | No violence — ambient speech, neutral turns | All NEU/NEG events |

??? info "ACOU vs DIST"
    `ACOU` captures **non-vocal acoustic cues** — a door slam, an object falling, an impact sound. These appear in Tier B clips as `background_events` in the `acoustic_scene` block.

    `DIST` captures **vocal distress** — screams, panic vocalisations, crying under coercion.

---

## Tier 2 subtypes (event-level)

| Tier 1 | Tier 2 subtype | Description |
|--------|----------------|-------------|
| VERB | `VERB_SHOUT` | Raised or shouted speech |
| VERB | `VERB_THREAT` | Direct verbal threats |
| VERB | `VERB_INSULT` | Demeaning or insulting language |
| DIST | `DIST_SCREAM` | Distress scream or panic vocalisation |
| DIST | `DIST_CRY` | Crying or sobbing under duress |
| PHYS | `PHYS_HARD` | Hard physical impact cue |
| PHYS | `PHYS_SOFT` | Softer physical contact cue |
| EMOT | `EMOT_GASLIGHT` | Gaslighting or reality-denial |
| EMOT | `EMOT_GUILT` | Guilt-tripping or emotional coercion |
| ACOU | `ACOU_SLAM` | Object slam or door slam |
| ACOU | `ACOU_FALL` | Object falling or thrown |
| NONE | `NONE_AMBIENT` | Regular ambient speech or neutral turn |

---

## Intensity scale (turn-level)

Intensity is scored 1–5 per dialogue turn. It controls prosody generation (pitch, rate, volume) and determines which tier1/tier2 labels are applied.

| Score | Label | Description | Prosody profile |
|-------|-------|-------------|----------------|
| 1 | Low tension | Calm conversation, mild undercurrent | Near-neutral |
| 2 | Moderate tension | Noticeable friction, raised voices | Slightly raised pitch/rate |
| 3 | Active conflict | Clear verbal aggression or intimidation | Elevated pitch, faster rate |
| 4 | Escalated violence | Physical or high-intensity verbal violence | High pitch, fast rate, volume up |
| 5 | Extreme / life-threatening | Severe physical violence, panic | Maximally expressive (capped) |

??? info "The prosody cap at I4–I5"
    At intensity 4–5, the LLM-generated prosody values are capped before SSML rendering to prevent Whisper transcription failures and maintain naturalness. The cap values are:

    - **Pitch:** max +2.0 semitones (post-cap)
    - **Rate:** range [0.85, 1.20] (post-cap)

    Any cap activation is recorded in `generation_metadata.effective_prosody_caps` per turn. You'll see many activations at I4–I5 in delivery-003 — this is expected. The cap was calibrated in a listening test in May 2026 (SynthBanshee PR #87).

---

## Distribution in delivery-003

| Typology | Clips | Projects | Tiers |
|----------|------:|---------|-------|
| SV | 5 | she_proves (3) + elephant (2) | A (3) + B (2) |
| IT | 5 | she_proves (3) + elephant (2) | A (3) + B (2) |
| NEG | 5 | she_proves (3) + elephant (2) | A (3) + B (2) |
| NEU | 5 | she_proves (3) + elephant (2) | A (3) + B (2) |

Intensity distribution across all 20 clips:

| Max intensity | Clips |
|:---:|:---:|
| 5 | 10 |
| 3 | 4 |
| 2 | 6 |
