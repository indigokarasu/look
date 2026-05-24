---
name: ocas-look
description: 'Look: image-to-action skill. Converts user-provided images into validated,
  decision-ready drafts across real-world action domains: events from flyers, macros
  from meal photos, places to save, products to price, receipts to log, documents
  to file, civic issues to report. Includes Google reverse image search for finding
  image sources, matches, and context. Trigger phrases: ''look at this image'', ''what
  is this'', ''scan this receipt'', ''what event is this'', ''how many calories'',
  ''save this place'', ''reverse image search this'', ''find this image'', ''update look''. Do not use for generic OCR, computer vision
  research, or surveillance.

  '
license: MIT
metadata:
  author: Indigo Karasu
  version: 2.5.1
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
9. Emit Signal files for extracted entities (places, events, products) to the `signal` payload field in the journal entry. Use Signal schema from `spec-ocas-shared-schemas.md`. Signal schema: Signal from spec-ocas-shared-schemas.md, with payload.type set to the ontology type of the primary entity and source_journal_type: "Observation".
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
- `look.reverse_search` — perform Google reverse image search on an image URL or local file. Returns matching pages, titles, and similar image URLs.

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

## Reverse Image Search

Look can perform reverse image search to find the source, context, or matches for a given image. This is useful for: identifying people in photos, finding product sources, locating event flyers, verifying image authenticity, and discovering higher-resolution versions.

### Primary: Yandex Images (browser-based)

**Yandex reverse image search works reliably from cloud IPs** where Google blocks. This is the preferred method.

**Workflow:**
1. Upload the image to a public host (see "Hosting local images" below)
2. Navigate browser to: `https://yandex.com/images/search?url=<encoded_image_url>&rpt=imageview`
3. Extract results from the page snapshot — Yandex renders results server-side, so `browser_snapshot` captures them directly
4. Key regions in the snapshot:
   - `"Image appears to contains"` — tags/descriptions of the image content
   - `"Similar images"` — visually similar images with source links
   - `"Sites with information about the image"` — web pages containing the image
   - `"In other sizes"` — available resolutions

**Extracting result URLs from snapshot:** Parse the snapshot text for link titles and URLs. Yandex results are plain HTML links, not JS-rendered.

### Fallback: `google-image-source-search` (Vorrik)

**Package:** `google-image-source-search` on PyPI. Pure Python, no API keys required. Scrapes Google's reverse image search results page.

**⚠️ CLOUD IP BLOCKING:** Google blocks image search uploads from cloud/server IPs. Both `search()` and `search_by_file()` will fail when run from a VPS/cloud environment. **Do not waste time retrying — go straight to Yandex.**

**Installation & usage:** See `references/pip-venv.md` (correct venv install pattern) and `references/google-image-source-search.md` (full usage reference).

### Hosting local images for reverse search

When you have a local file and need a public URL for reverse image search:

**Imgur anonymous upload (most reliable):**
See `references/credential-files.md` for the Imgur client ID and upload code pattern.

Other hosts (0x0.st, transfer.sh) are frequently down or slow. Imgur is the most reliable.

### Decision tree

```
Have image (local file or URL)
  │
  ├─ Is it a local file?
  │   ├─ YES → Upload to Imgur first → get public URL
  │   └─ NO (already a URL) → Use directly
  │
  └─ Reverse search via Yandex (browser)
      ├─ Results found → Done
      └─ No results → Try Google (only if on residential IP)
```

**Integration with core workflow:** Reverse image search runs automatically during step 5 (Research and validate externally) when the ingested image is a photo of a person, product, place, or document. Results feed into entity extraction and draft generation.

**Pitfalls:**
- **Google blocks cloud IPs completely.** From any VPS/cloud server, Google reverse image search will fail. Use Yandex instead. Do not retry Google more than once.
- `search_by_file()` uploads the image directly to Google — fails from cloud IPs.
- `search()` passes the URL to Google — also fails from cloud IPs (Google checks the request origin, not just the image host).
- Yandex results are in Russian sometimes — the snapshot text may contain Cyrillic tags. The site URLs and image URLs are still usable.
- Imgur's public client ID has rate limits. See `references/credential-files.md` for details.
- TinEye returns JS-rendered results that can't be scraped from curl — use the browser if TinEye is needed.

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

1. Read `source:` from frontmatter → extract `{owner}/{repo}` from URL
2. Read local version from SKILL.md frontmatter `metadata.version`
3. Fetch remote version from SKILL.md frontmatter: `gh api "repos/{owner}/{repo}/contents/SKILL.md" --jq '.content' | base64 -d | grep 'version:' | head -1 | sed 's/.*"\(.*\)".*/\1/'`
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

## Gotchas

- **Google reverse image search is blocked from cloud IPs** — Use Yandex Images via browser instead. Do not retry Google more than once.
- **EXIF location ≠ event venue** — The capture location embedded in a photo's EXIF data is where the photo was taken, not necessarily the event venue.
- **Never invent OCR text, barcodes, prices, or license plates** — If the image content is ambiguous, state uncertainty rather than guessing.
- **TinEye returns JS-rendered results** — Use the browser if TinEye is needed; it can't be scraped with curl.
- **Reverse image search requires a public URL** — Upload local files to Imgur first, then use the returned URL for Yandex reverse search.

## Support file map

| File | When to read |
|------|-------------|
| `references/schemas.md` | Before creating evidence, drafts, or receipt records; when validating data structures |
| `references/domain_playbooks.md` | Before domain routing or draft generation; when checking per-domain behavior |
| `references/decision_policy.md` | Before risk assessment or confirmation decisions; when classifying action risk level |
| `references/command_reference.md` | Before any command execution; when checking command syntax or parameters |
| `references/storage_and_config.md` | Before config changes or storage operations; when modifying skill state |
| `references/journal.md` | Before calling look.journal; at end of every run |
| `references/pip-venv.md` | Before installing any pip package for a skill; when setting up Python dependencies |

## Update command

This skill self-updates every 24 hours via:

```bash
look.update
```

This pulls the latest version from GitHub and restarts the skill's background tasks if applicable.
