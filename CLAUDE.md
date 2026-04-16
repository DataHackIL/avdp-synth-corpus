# avdp-synth-corpus — Claude Code Context

## What this repo is

**Data-only repository.** No application code lives here. All pipeline logic, configuration,
templates, and tests are in [DataHackIL/SynthBanshee](https://github.com/DataHackIL/SynthBanshee).

Contents:
- `data/he/` — spec-compliant generated clips (`.wav` + `.txt` + `.json` + `.jsonl` per clip)
- `assets/speech/` — per-utterance TTS WAV cache (SHA-256 keyed)
- `assets/scripts/` — per-scene LLM script cache (SHA-256 keyed)
- `deliveries/` — per-delivery metadata and notes
- `DELIVERIES.md` — master delivery log

---

## Hard rules — read before touching any file

### Cache integrity (assets/)

`assets/speech/` and `assets/scripts/` files are named by **SHA-256 of their generation inputs**.
SynthBanshee uses the filename as the cache key.

- **Never rename, modify, or reformat** a file in `assets/`. Renaming breaks cache lookups silently.
- **Never delete** a cache file. Deletion forces a paid Azure TTS re-synthesis call.
- **Never add** a file with an incorrect name. A wrong hash name silently poisons the cache.
- The only safe operations are: add a correctly-named file (produced by SynthBanshee), or leave it alone.

### Clip file integrity (data/)

- Every `.wav` must have a matching `.txt`, `.json`, and `.jsonl` with the same stem.
- **Never edit `.wav` files by hand.** All audio is produced by `synthbanshee generate` or
  `synthbanshee generate-batch`. Manual edits silently break the SHA-256 cache linkage in
  `assets/speech/`.
- **ASCII-only filenames**, lowercase, no spaces. No Hebrew text in filenames or JSON
  keys/values. Hebrew belongs in `.txt` transcript files only.
- `is_synthetic: true` must be present in all `.json` clip metadata.

### Delivery log

Every merged PR that adds clips must have:
- A row in `DELIVERIES.md`
- A `deliveries/{NNN-slug}/metadata.yaml` and `deliveries/{NNN-slug}/notes.md`

Do not merge data without updating the delivery log.

---

## Label policy

### `has_violence` is a derived convenience field — keep it

`has_violence` appears in `weak_label.has_violence` (clip `.json`) and as a column in
`manifest.csv`. It is **derived** from the hierarchical taxonomy, not assigned independently:

```
has_violence = (violence_typology in {"SV", "IT", "NEG"}) and (max_intensity >= 3)
```

**Do not remove it.** AI teams use it for binary baseline models and stratified train/val/test
splits. Removing it forces every downstream user to re-derive it from `violence_typology` and
`max_intensity` — which introduces inconsistency risk.

### The taxonomy is the ground truth

The label hierarchy (in `configs/taxonomy.yaml` in SynthBanshee) is the authoritative source:

| Level | Field | Examples |
|-------|-------|---------|
| Violence typology | `violence_typology` | `SV`, `IT`, `NEG`, `NEU` |
| Tier 1 category | `tier1_category` | `PHYS`, `VERB`, `DIST`, `ACOU`, `EMOT`, `NONE` |
| Tier 2 subtype | `tier2_subtype` | `VERB_THREAT`, `DIST_SCREAM`, `PHYS_HARD` |

**What is prohibited:** replacing the full taxonomy with only a binary flag. `has_violence` must
always appear alongside `violence_typology`, `tier1_category`, `tier2_subtype`, and
`max_intensity` — never as a substitute for them.

---

## Audio format (all clips must conform)

| Property | Value |
|----------|-------|
| Sample rate | 16 kHz |
| Channels | Mono |
| Bit depth | 16-bit PCM WAV |
| Peak level | −1.0 dBFS (peak-normalized) |
| Silence padding | ≥ 0.5 s before and after target speech |
| Language | Hebrew (he-IL) |

---

## How clips get here

SynthBanshee routes output here via three environment variables:

| Variable | Points to |
|----------|-----------|
| `SYNTHBANSHEE_DATA_DIR` | `data/he/` |
| `SYNTHBANSHEE_CACHE_DIR` | `assets/speech/` |
| `SYNTHBANSHEE_SCRIPT_CACHE_DIR` | `assets/scripts/` |

Run `synthbanshee generate-batch -r <run_config.yaml>` from the SynthBanshee repo with these
variables set. Never write clip files to this repo by hand.

---

## Validation

From the SynthBanshee repo:

```bash
# Single clip
synthbanshee validate data/he/{speaker_id}/{clip_id}.wav

# Whole directory
synthbanshee qa-report data/he/
```

A clip is valid iff all four files are present, the WAV is spec-compliant, the `.json` parses
as `ClipMetadata` with `is_synthetic=true`, and the filename stem is ASCII lowercase.

---

## What NOT to do

- Don't modify or rename files in `assets/` — cache integrity depends on exact filenames
- Don't edit `.wav` files manually
- Don't add clips without updating `DELIVERIES.md` and `deliveries/{slug}/`
- Don't use Hebrew text in filenames or JSON string fields
- Don't drop `has_violence` from metadata or manifests
- Don't treat `has_violence` as the sole label — always preserve the full taxonomy alongside it
- Don't use lossy audio formats (MP3, AAC) anywhere
- Don't commit `.env` files or credentials
