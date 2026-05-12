# Label Taxonomy

Three levels: clip-level **typology** → event-level **tier 1 category** → event-level **tier 2 subtype**. Plus a per-turn **intensity** (1–5) that drives prosody. The source of truth is `taxonomy.yaml` in [SynthBanshee](https://github.com/DataHackIL/SynthBanshee).

---

## Violence typology (clip-level)

The `violence_typology` field. **Not** an ordered scale.

| Code | Name | What it sounds like |
|------|------|---------------------|
| `SV` | Severe Violence | Physical violence, life-threatening escalation. `tier1_category` includes `PHYS`, `DIST`, often `VERB`. |
| `IT` | Intimate Terrorism | Sustained coercive control, repeated verbal/emotional abuse — typically without physical attack. Heavy on `VERB` and `EMOT`. |
| `NEG` | Negative confusor | Acoustically intense but non-violent — anger, argument, distress, crying. **Hard negative class.** All events are `tier1_category: "NONE"`. |
| `NEU` | Neutral | Calm or mundane conversation. No violence markers. |

!!! danger "NEG is the trap"
    A NEG clip can have raised voices, crying, and `max_intensity: 3`. It will *sound* like violence to a model that only listens for loudness or emotional tone. But it is by definition `has_violence: false` — its purpose is to teach your model the difference between distress and violence. Training NEG as a positive class will collapse your precision. See [Gotcha #5](gotchas.md#5-neg-is-not-violent-at-low-intensity).

---

## `has_violence` — derived from events

```python
has_violence = any(e["tier1_category"] != "NONE" for e in events)
```

That's the rule. Two consequences worth knowing:

- **NEG clips are always `has_violence: false`** — every event in a NEG clip has `tier1_category: "NONE"` by construction, even when `max_intensity` is high.
- **NEU clips are always `has_violence: false`** for the same reason.
- A `SV` or `IT` clip is `has_violence: true` because at least one event has a non-NONE category.

!!! danger "Don't derive `has_violence` from typology or intensity"
    ```python
    has_violence = typology in ("SV", "IT")     # WRONG — works on this corpus but fragile
    has_violence = max_intensity >= 3           # VERY WRONG — fires on every NEG clip
    ```
    The event-level taxonomy is the ground truth. `weak_label.has_violence` exists for fast filtering and baseline modelling only — never as the sole training label. Train on the strong-label events when you can.

---

## Tier 1 categories (event-level)

The `tier1_category` field on each `EventLabel`. Six values.

| Category | What it covers | Where it shows up |
|----------|----------------|-------------------|
| `VERB` | Verbal violence — threats, shouting, demeaning language | Most violent clips, all intensity levels |
| `DIST` | Distress vocalisations — screaming, crying under duress | I3+ turns in SV/IT, peak escalation |
| `PHYS` | Physical violence cues — impact sounds, struggle | I4+ turns in SV clips |
| `EMOT` | Emotional manipulation — gaslighting, guilt-tripping | IT clips, coercive control turns |
| `ACOU` | Acoustic non-vocal events — slams, falls | Tier B clips, recorded in `acoustic_scene.background_events` |
| `NONE` | Ambient speech / neutral turn | All NEU clips, all NEG clips, calm turns in SV/IT |

!!! info "ACOU vs DIST"
    `ACOU` is **non-vocal** acoustic — a door slam, an object hitting the floor. `DIST` is **vocal distress** — a scream, crying. A scene where someone throws a glass and the victim screams will have an `ACOU_SLAM` event for the glass and a `DIST_SCREAM` event for the scream.

    Tier B Elephant clips inject `ACOU_*` events as part of room augmentation; they show up both in `acoustic_scene.background_events` (with audio-level metadata) and in `.jsonl` strong labels (as labelled events). Tier A She-Proves clips can't produce ACOU events — there's no room-augmentation stage to add them.

---

## Tier 2 subtypes (event-level)

| Tier 1 | Tier 2 | Description |
|--------|--------|-------------|
| `VERB` | `VERB_SHOUT` | Raised or shouted speech |
| `VERB` | `VERB_THREAT` | Direct verbal threats |
| `VERB` | `VERB_INSULT` | Demeaning or insulting language |
| `DIST` | `DIST_SCREAM` | Distress scream or panic vocalisation |
| `DIST` | `DIST_CRY` | Crying or sobbing under duress |
| `PHYS` | `PHYS_HARD` | Hard physical impact cue |
| `PHYS` | `PHYS_SOFT` | Softer physical contact cue |
| `EMOT` | `EMOT_GASLIGHT` | Gaslighting or reality-denial |
| `EMOT` | `EMOT_GUILT` | Guilt-tripping or emotional coercion |
| `ACOU` | `ACOU_SLAM` | Object slam or door slam |
| `ACOU` | `ACOU_FALL` | Object falling or thrown |
| `NONE` | `NONE_AMBIENT` | Regular ambient speech or neutral turn |

---

## Intensity scale (turn-level)

Each turn has an `intensity` in `[1, 5]`. It controls prosody generation (pitch, rate, volume) and the LLM script tone.

| Score | Label | What's happening |
|-------|-------|------------------|
| 1 | Low tension | Calm conversation, mild undercurrent |
| 2 | Moderate tension | Noticeable friction, raised voices |
| 3 | Active conflict | Clear verbal aggression or intimidation |
| 4 | Escalated violence | Physical or high-intensity verbal violence |
| 5 | Extreme | Severe physical violence, panic, imminent danger |

### How intensity and typology relate

They are correlated but not the same.

| Typology | Typical `max_intensity` range | Why |
|----------|:-----------------------------:|-----|
| `NEU` | 1–2 | Mundane conversation by definition |
| `NEG` | 2–3 | Distressed but non-violent; intensity rises with shouting/crying, but no PHYS/DIST events fire |
| `IT` | 3–5 | Sustained verbal/emotional aggression; can hit I5 on threats without physical violence |
| `SV` | 4–5 | Physical escalation requires I4+ turns |

In delivery-003 the actual distribution is `max_intensity` 5 = 10 clips, 3 = 4 clips, 2 = 6 clips. Useful for designing stratified eval splits: if you want a balanced eval set across intensity *and* typology, you'll need to upsample (or wait for more data).

??? info "What is the prosody cap?"
    At I3+, the LLM-suggested prosody values are clamped before SSML rendering to keep speech natural and transcribable by Whisper:

    - **Pitch:** capped at +2.0 semitones (post-cap)
    - **Rate:** clamped to [0.85, 1.20]

    When clamping fires, the pre- and post-cap values are recorded per turn in `generation_metadata.effective_prosody_caps`. You'll see many activations at I4–I5 in delivery-003 — that's the intended behaviour, calibrated by listening test in May 2026 (SynthBanshee PR #87).

---

## Where the labels come from

- **Strong labels (`.jsonl`)** are generated by SynthBanshee from the LLM-authored script — the LLM produces turn-level intensity and an action tag (`VERB_SHOUT`, `DIST_SCREAM`, …), SynthBanshee converts them into `EventLabel` records.
- **Weak labels (`.json` → `weak_label`)** are derived from the strong labels by aggregation.
- **No human annotation has happened.** `confidence` is always `1.0`; `label_source` is always `"auto"`; `iaa_reviewed` is always `false`. Future deliveries may introduce human review on a subset — they'll set `iaa_reviewed: true` per clip when that happens.

The scripts themselves are LLM-generated Hebrew dialogue, conditioned on the scene YAML and persona definitions in SynthBanshee. They are **not** transcripts of real conversations.

---

## Distribution in delivery-003

| Typology | Clips | Tier A (she_proves) | Tier B (elephant) |
|----------|------:|:-------------------:|:------------------:|
| `SV` | 5 | 3 | 2 |
| `IT` | 5 | 3 | 2 |
| `NEG` | 5 | 3 | 2 |
| `NEU` | 5 | 3 | 2 |

Balanced across typology and across project. Not balanced across speakers — see [Deliveries](deliveries.md#known-limitations).
