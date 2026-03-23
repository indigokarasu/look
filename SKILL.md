---
name: ocas-look
description: Look: image-to-action skill. Converts user-provided images into validated, decision-ready drafts across real-world action domains: events from flyers, macros from meal photos, places to save, products to price, receipts to log, documents to file, civic issues to report. Trigger phrases: 'look at this image', 'what is this', 'scan this receipt', 'what event is this', 'how many calories', 'save this place'. Do not use for generic OCR, computer vision research, or surveillance.
metadata: {"openclaw":{"emoji":"👁️"}}
---

# Look

Look converts images into validated, decision-ready action drafts. It infers intent, extracts evidence, researches and validates, reduces ambiguity before asking questions, and produces drafts requiring explicit confirmation for execution.

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


## Functions

### look_ingest_image()

**Purpose:** ingest image(s) with optional EXIF and device pre-parse

**Returns:** Operation result

### look_propose_actions()

**Purpose:** generate ActionDrafts with DecisionRecords

**Returns:** Operation result

### look_execute_action()

**Purpose:** execute a confirmed draft (requires explicit approval)

**Returns:** Operation result

### look_rollback_action()

**Purpose:** attempt rollback for reversible actions

**Returns:** Operation result

### look_status()

**Purpose:** last ingest, pending drafts, items awaiting confirmation

**Returns:** Operation result

### look_config_set()

**Purpose:** update configuration

**Returns:** Operation result

### look_journal()

**Purpose:** write journal for the current run; called at end of every run

**Returns:** Operation result



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
9. Emit Signal files for extracted entities (places, events, products) to `~/openclaw/db/ocas-elephas/intake/{signal_id}.signal.json`. Use Signal schema from `spec-ocas-shared-schemas.md`.
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
~/openclaw/data/ocas-look/
  config.json
  state.json
  events.jsonl
  decisions.jsonl
  reports/
  artifacts/

~/openclaw/journals/ocas-look/
  YYYY-MM-DD/
    {run_id}.json
```


Default config.json:
```json
{
  "skill_id": "ocas-look",
  "skill_version": "2.2.0",
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

1. Create `~/openclaw/data/ocas-look/` and subdirectories (`reports/`, `artifacts/`)
2. Write default `config.json` and `state.json` if absent
3. Create empty JSONL files: `events.jsonl`, `decisions.jsonl`
4. Create `~/openclaw/journals/ocas-look/`
5. Ensure `~/openclaw/db/ocas-elephas/intake/` exists (create if missing)
6. Log initialization as a DecisionRecord in `decisions.jsonl`

Look is purely reactive. No cron jobs or heartbeat entries.

## Visibility

public

## Support file map

File | When to read
`references/schemas.md` | Before creating evidence, drafts, or receipts
`references/domain_playbooks.md` | Before domain routing or draft generation
`references/decision_policy.md` | Before risk assessment or confirmation decisions
`references/command_reference.md` | Before any command execution
`references/storage_and_config.md` | Before config changes or storage operations
`references/journal.md` | Before look.journal; at end of every run
