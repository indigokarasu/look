---
name: ocas-look
description: >
  Converts user-provided images into validated, decision-ready action drafts.
  Routes images by inferred intent across domains: events from flyers, macros from
  meal photos, places to save, products to price, receipts to log, documents to file,
  civic issues to report. Supports reverse image search via Yandex (browser) and
  Google (residential IP fallback).
  Trigger: flyer, receipt, meal photo, product photo, document scan, civic issue photo,
  "look at this image", "what is this", "scan this", "reverse image search".
  NOT for generic OCR, computer vision research, or surveillance.
license: MIT
source: https://github.com/indigokarasu/look
includes:
  - references/**
  - scripts/**
metadata:
  author: Indigo Karasu
  version: 2.5.1
---
## When to Use

- Image analysis and visual content extraction
- Reverse image search and source identification
- Visual verification of web content
- Screenshot analysis and OCR
- When any skill needs image understanding
## When NOT to Use

- Image generation (use Imagine)
- Video processing or analysis
- Bulk image processing at scale
- Real-time camera feed analysis

# Look

Look bridges the physical world and the digital agent — it takes a user-provided image, infers what the user probably wants done with it, and produces a validated, decision-ready action draft across domains including calendar events, meal macros, places, product comparisons, receipts, documents, and civic reports. It resolves ambiguity through research and option reduction before asking any clarifying questions, and nothing executes without explicit per-draft confirmation.

## When to use

- A flyer or poster → calendar event or ticket purchase draft
- A meal photo → macro estimation
- A storefront or sign → save to try-list
- A product photo → pricing comparison or order draft
- A civic issue photo → 311 report draft
- A receipt → expense entry
- A document → searchable PDF and filing draft

## When NOT to use

- Generic OCR utility (use a dedicated OCR skill)
- Universal computer vision toolkit (use a CV-specific tool)
- Background surveillance or tracking
- Generic automation framework
- Images where the user has no clear intent or action in mind

## What this skill does not do

- Generic OCR utility
- Universal computer vision toolkit
- Background surveillance or tracking
- Generic automation framework

## Responsibility boundary

Look owns image-to-action conversion: ingest, context inference, domain routing, draft generation, and execution with confirmation.

Look does not own: web research (Sift), knowledge graph writes (Elephas), preference persistence (Taste), communications (Dispatch).

## Ontology types

Look works with these types from `spec-ocas-ontology.md`:

- **Entity/Person** — people identified in images (public figures, named contacts).
- **Place** — venues, locations, and scenes extracted from image context.
- **Concept/Event** — events visible in images (gatherings, occasions, dated scenes).
- **Thing/DigitalArtifact** — the source image itself.

Look emits Signals to Elephas using the Signal schema from `spec-ocas-shared-schemas.md`. The `payload.type` field must be set to the ontology type of the primary extracted entity (`Person`, `Place`, `Event`, or `DigitalArtifact`). `source_journal_type` is `"Observation"`.

## Supported domains

Events, food macros, places, products, civic issues, receipts, documents.

Read `references/domain_playbooks.md` for detailed per-domain behavior.

## Core workflow

See `references/workflow.md` for the full step-by-step procedure, error handling table, and clarification rules.

## Commands

- `look.ingest.image` — ingest image(s) with optional EXIF and device pre-parse
- `look.propose.actions` — generate ActionDrafts with DecisionRecords
- `look.execute.action` — execute a confirmed draft (requires explicit approval)
- `look.rollback.action` — attempt rollback for reversible actions
- `look.status` — last ingest, pending drafts, items awaiting confirmation
- `look.config.set` — update configuration
- `look.journal` — write journal for the current run; called at end of every run
- `look.update` — pull latest from GitHub source; preserves journals and data
- `lookup.reverse_search` — perform Google reverse image search on an image URL or local file. Returns matching pages, titles, and similar image URLs.

## Confirmation and rollback rules

- Draft-first always. No execution without explicit confirmation.
- High-risk actions (purchases, 311 submission, health writes): require per-draft confirmation token.
- Reversible actions (calendar, maps): expose rollback information.

## Permission discipline

Default deny. Request minimally. Drafting continues even without execution permissions. Blocked execution reported, not silently skipped.

## Boundaries

- Never invent OCR text, barcodes, prices, or license plates
- EXIF capture location is not the event venue
- iOS relay pre-parse is optional evidence, not truth
- The skill must work without relay upload

## Error handling

See `references/workflow.md` for the full error handling table.

## Reverse Image Search

Look uses reverse image search during step 5 (Research and validate externally). Primary method: Yandex Images via browser (works from cloud IPs). Fallback: `google-image-source-search` PyPI package (residential IPs only).

**Quick reference:**
- Local file → upload to Imgur first → public URL → Yandex search
- Already a public URL → use directly with Yandex
- Google fallback only on residential IP — never retry more than once from cloud

Detailed instructions: `references/google-image-source-search.md` (Google fallback), `references/credential-files.md` (Imgur client ID).

**Gotchas (also in Gotchas section):**
- Google blocks cloud IPs completely — use Yandex
- TinEye returns JS-rendered results — use browser
- Yandex results may contain Cyrillic tags — URLs still usable

## Storage layout

```
{agent_root}/commons/data/ocas-look/
  config.json
  state.json
  events.jsonl
  decisions.jsonl
  reports/
  artifacts/

{agent_root}/commons/journals/ocas-look/
  YYYY-MM-DD/
    {run_id}.json
```

Default config: see `references/default_config.md`.

## OKRs

See `references/okrs.md` for the full OKR definitions. Universal OKRs from spec-ocas-journal.md apply to all runs.

## Optional skill cooperation

- Sift — web research for validation during draft generation (via SearchX)
- Elephas — emit Signal files for extracted entities after draft generation

## Journal outputs

Observation Journal — all image ingestion and draft generation runs.

## Initialization

On first invocation of any Look command, run `look.init`:

1. Create `{agent_root}/commons/data/ocas-look/` and subdirectories (`reports/`, `artifacts/`)
2. Write default `config.json` and `state.json` if absent
3. Create empty JSONL files: `events.jsonl`, `decisions.jsonl`
4. Create `{agent_root}/commons/journals/ocas-look/`
5. Ensure journal payload fields (see interfaces specification) exists (create if missing)
6. Register cron job `look:update` if not already present (check the platform scheduling registry first)
7. Log initialization as a DecisionRecord in `decisions.jsonl`
8. **Reverse image search setup** (run once):
   - Read `references/pip-venv.md` for the full setup pattern (install `google-image-source-search`, pin `requests`, verify import)

## Background tasks

| Job name | Mechanism | Schedule | Command |
|---|---|---|---|
| `look:update` | cron | `0 0 * * *` (midnight daily) | `look.update` |

```
# Task declared in SKILL.md frontmatter metadata.{platform}.cron
```

## Self-update

`look.update` pulls the latest package from the `source:` URL in this file's frontmatter. Runs silently — no output unless the version changed or an error occurred.

See `references/self_update.md` for the full self-update procedure.

## Visibility

public

## Gotchas

- **Google reverse image search is blocked from cloud IPs** — Use Yandex Images via browser instead. Do not retry Google more than once.
- **EXIF location ≠ event venue** — The capture location embedded in a photo's EXIF data is where the photo was taken, not necessarily the event venue.
- **Never invent OCR text, barcodes, prices, or license plates** — If the image content is ambiguous, state uncertainty rather than guessing.
- **TinEye returns JS-rendered results** — Use the browser if TinEye is needed; it can't be scraped with curl.
- **Reverse image search requires a public URL** — Upload local files to Imgur first, then use the returned URL for Yandex reverse search.

## Support File Map

| File | When to read |
|------|-------------|
| `references/schemas.md` | Before creating evidence, drafts, or receipt records; when validating data structures |
| `references/domain_playbooks.md` | Before domain routing or draft generation; when checking per-domain behavior |
| `references/decision_policy.md` | Before risk assessment or confirmation decisions; when classifying action risk level |
| `references/command_reference.md` | Before any command execution; when checking command syntax or parameters |
| `references/storage_and_config.md` | Before config changes or storage operations; when modifying skill state |
| `references/journal.md` | Before calling look.journal; at end of every run |
| `references/pip-venv.md` | Before installing any pip package for a skill; when setting up Python dependencies |
| `references/google-image-source-search.md` | Before using Google reverse image search fallback; when on residential IP and Yandex is unavailable |
| `references/credential-files.md` | Before reverse image search with local files; when Imgur client ID is needed |
| `references/default_config.md` | Before initialization or config changes; when referencing default config values |
| `references/okrs.md` | Before writing or evaluating OKRs; when checking skill-level targets |
| `references/workflow.md` | Before executing the core workflow; when handling errors during processing |
| `references/self_update.md` | Before running `look.update`; when debugging self-update failures |

## Update command

This skill self-updates every 24 hours via:

```bash
look.update
```

This pulls the latest version from GitHub and restarts the skill's background tasks if applicable.
