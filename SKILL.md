---
name: ocas-look
description: >
  Look: image-to-action skill. Converts user-provided images into validated,
  decision-ready drafts across real-world action domains: events from flyers,
  macros from meal photos, places to save, products to price, receipts to log,
  documents to file, civic issues to report. Trigger phrases: 'look at this
  image', 'what is this', 'scan this receipt', 'what event is this', 'how many
  calories', 'save this place', 'update look'. Do not use for generic OCR,
  computer vision research, or surveillance.
metadata:
  author: Indigo Karasu
  email: mx.indigo.karasu@gmail.com
  version: "2.4.0"
  hermes:
    tags: [images, ocr, visual]
    category: signal
    cron:
      - name: "look:update"
        schedule: "0 0 * * *"
        command: "look.update"
  openclaw:
    skill_type: system
    visibility: public
    filesystem:
      read:
        - "$OCAS_DATA_ROOT/data/ocas-look/"
        - "$OCAS_DATA_ROOT/journals/ocas-look/"
      write:
        - "$OCAS_DATA_ROOT/data/ocas-look/"
        - "$OCAS_DATA_ROOT/journals/ocas-look/"
        - "$OCAS_DATA_ROOT/db/ocas-elephas/intake/"
    self_update:
      source: "https://github.com/indigokarasu/look"
      mechanism: "version-checked tarball from GitHub via gh CLI"
      command: "look.update"
      requires_binaries: [gh, tar, python3]
    cron:
      - name: "look:update"
        schedule: "0 0 * * *"
        command: "look.update"
---

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

1. Ingest image(s) with EXIF if available
2. Merge extraction evidence (OCR, entities, context)
3. Infer context and likely intent
4. Route to appropriate domain
5. Research and validate externally
6. Filter by constraints (dietary, preferences, permissions)
7. Reduce options before asking questions
8. Generate 1-3 decision-ready ActionDrafts
9. Emit Signal files for extracted entities (places, events, products) to `$OCAS_DATA_ROOT/db/ocas-elephas/intake/{signal_id}.signal.json`. Use Signal schema from `spec-ocas-shared-schemas.md`. Signal schema: Signal from spec-ocas-shared-schemas.md, with payload.type set to the ontology type of the primary entity and source_journal_type: "Observation".
10. Write journal via `look.journal`

Clarification happens only after option reduction, not before.

## Commands

- `look.ingest.image` — ingest image(s) with optional EXIF and device pre-parse
- `look.propose.actions` — generate ActionDrafts with DecisionRecords
- `look.execute.action` — execute a confirmed draft (requires explicit approval)
- `look.rollback.action` — attempt rollback for reversible actions
- `look.status` — last ingest, pending drafts, items awaiting confirmation
- `look.config.set` — update configuration
- `look.journal` — write journal for the current run; called at end of every run
- `look.update` — pull latest from GitHub source; preserves journals and data

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

## Storage layout

```
$OCAS_DATA_ROOT/data/ocas-look/
  config.json
  state.json
  events.jsonl
  decisions.jsonl
  reports/
  artifacts/

$OCAS_DATA_ROOT/journals/ocas-look/
  YYYY-MM-DD/
    {run_id}.json
```


Default config.json:
```json
{
  "skill_id": "ocas-look",
  "skill_version": "2.3.0",
  "config_version": "1",
  "created_at": "",
  "updated_at": "",
  "domains": {
    "events": true,
    "food": true,
    "places": true,
    "products": true,
    "civic": true,
    "receipts": true,
    "documents": true
  },
  "user_profile": {
    "diet": "vegetarian"
  },
  "commerce": {
    "auto_purchase": false
  },
  "retention": {
    "days": 30,
    "max_records": 10000
  }
}
```

## OKRs

Universal OKRs from spec-ocas-journal.md apply to all runs.

```yaml
skill_okrs:
  - name: draft_accuracy
    metric: fraction of ActionDrafts accepted without modification
    direction: maximize
    target: 0.75
    evaluation_window: 30_runs
  - name: domain_routing_accuracy
    metric: fraction of images routed to correct domain on first attempt
    direction: maximize
    target: 0.90
    evaluation_window: 30_runs
  - name: confirmation_compliance
    metric: fraction of high-risk actions requiring confirmation before execution
    direction: maximize
    target: 1.0
    evaluation_window: 30_runs
```

## Optional skill cooperation

- Sift — web research for validation during draft generation
- Elephas — emit Signal files for extracted entities after draft generation

## Journal outputs

Observation Journal — all image ingestion and draft generation runs.

## Initialization

On first invocation of any Look command, run `look.init`:

1. Create `$OCAS_DATA_ROOT/data/ocas-look/` and subdirectories (`reports/`, `artifacts/`)
2. Write default `config.json` and `state.json` if absent
3. Create empty JSONL files: `events.jsonl`, `decisions.jsonl`
4. Create `$OCAS_DATA_ROOT/journals/ocas-look/`
5. Ensure `$OCAS_DATA_ROOT/db/ocas-elephas/intake/` exists (create if missing)
6. Register cron job `look:update` if not already present (check the platform scheduling registry first)
7. Log initialization as a DecisionRecord in `decisions.jsonl`

## Background tasks

| Job name | Mechanism | Schedule | Command |
|---|---|---|---|
| `look:update` | cron | `0 0 * * *` (midnight daily) | `look.update` |

```
# Task declared in SKILL.md frontmatter metadata.{platform}.cron
```


## Self-update

`look.update` pulls the latest package from the `source:` URL in this file's frontmatter. Runs silently — no output unless the version changed or an error occurred.

1. Read `source:` from frontmatter → extract `{owner}/{repo}` from URL
2. Read local version from `skill.json`
3. Fetch remote version: `gh api "repos/{owner}/{repo}/contents/skill.json" --jq '.content' | base64 -d | python3 -c "import sys,json;print(json.load(sys.stdin)['version'])"`
4. If remote version equals local version → stop silently
5. Download and install:
   ```bash
   TMPDIR=$(mktemp -d)
   gh api "repos/{owner}/{repo}/tarball/main" > "$TMPDIR/archive.tar.gz"
   mkdir "$TMPDIR/extracted"
   tar xzf "$TMPDIR/archive.tar.gz" -C "$TMPDIR/extracted" --strip-components=1
   cp -R "$TMPDIR/extracted/"* ./
   rm -rf "$TMPDIR"
   ```
6. On failure → retry once. If second attempt fails, report the error and stop.
7. Output exactly: `I updated Look from version {old} to {new}`

## Visibility

public

## Support file map

| File | When to read |
|---|---|
| `references/schemas.md` | Before creating evidence, drafts, or receipts |
| `references/domain_playbooks.md` | Before domain routing or draft generation |
| `references/decision_policy.md` | Before risk assessment or confirmation decisions |
| `references/command_reference.md` | Before any command execution |
| `references/storage_and_config.md` | Before config changes or storage operations |
| `references/journal.md` | Before look.journal; at end of every run |

## Update command

This skill self-updates every 24 hours via:

```bash
openclaw look.update
```

This pulls the latest version from GitHub and restarts the skill's background tasks if applicable.
