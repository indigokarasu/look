---
name: ocas-look
description: >
  Image-to-action skill. Converts user-provided images into validated,
  decision-ready drafts across real-world action domains.
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

Clarification happens only after option reduction, not before.

## Commands

- `look.ingest.image` — ingest image(s) with optional EXIF and device pre-parse
- `look.propose.actions` — generate ActionDrafts with DecisionRecords
- `look.execute.action` — execute a confirmed draft (requires explicit approval)
- `look.rollback.action` — attempt rollback for reversible actions
- `look.status` — last ingest, pending drafts, items awaiting confirmation
- `look.config.set` — update configuration

## Confirmation and rollback rules

- Draft-first always. No execution without explicit confirmation.
- High-risk actions (purchases, 311 submission, health writes): require per-draft confirmation token.
- Reversible actions (calendar, maps): expose rollback information.

## Permission discipline

Default deny. Request minimally. Drafting continues even without execution permissions. Blocked execution reported, not silently skipped.

## Support file map

- `references/schemas.md` — ImageEvidence, ContextModel, ActionDraft, ExecutionReceipt, DecisionRecord
- `references/domain_playbooks.md` — per-domain rules for events, food, places, products, civic, receipts, documents
- `references/decision_policy.md` — risk tiers, confirmation rules, rollback, no-hallucination policy
- `references/command_reference.md` — detailed command inputs, outputs, and decision stages
- `references/storage_and_config.md` — storage layout, config fields, canonical config example

## Storage layout

```
.look/
  config.json
  state.json
  events.jsonl
  decisions.jsonl
  reports/
  artifacts/
```

## Boundaries

- Never invent OCR text, barcodes, prices, or license plates
- EXIF capture location is not the event venue
- iOS relay pre-parse is optional evidence, not truth
- The skill must work without relay upload
