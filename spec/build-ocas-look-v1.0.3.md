# Build Specification for `ocas-look` v1.0.3

## Build summary
- Skill name: `ocas-look`
- Package name: `ocas-look`
- Version: `1.0.3`
- Skill type: `workflow`
- Author: `Indigo Karasu`
- Email: `mx.indigo.karasu@gmail.com`
- Build objective: Build a complete, ready-to-install Agent Skill that converts user-provided images into validated, evidence-backed, decision-ready drafts across a bounded set of real-world action domains, performs aggressive preprocessing before asking questions, and requires explicit confirmation before any state-changing action.

## External standard
Build this as a real Anthropic-style Agent Skill package. Optimize for:
- strong routing metadata and realistic trigger phrasing
- concise but operational `SKILL.md`
- progressive disclosure through support files for schemas, playbooks, and policy details
- deterministic behavior where safety or reversibility matters

Reference standard for packaging and authoring approach:
- Anthropic Agent Skills best practices: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices

## Product to generate
Return the complete installable skill package, not a planning memo and not a build handoff.

The final package must include every file required to install and use the skill.

## Skill purpose
`ocas-look` is an image-to-action skill.

It activates when the user provides one or more images and wants the system to:
- infer likely intent from what is visible
- extract structured evidence
- research and validate likely interpretations
- reduce ambiguity before asking follow-up questions
- produce one or more decision-ready drafts
- execute only after explicit confirmation for state-changing actions

This skill is not:
- a generic OCR utility
- a universal computer-vision toolkit
- a background surveillance or tracking system
- a general automation framework
- a generic gallery organizer

## Functional surface to preserve
The built skill must preserve the full functional surface below. Do not silently drop capabilities because of package cleanup or concision.

Supported domains:
- events and flyers
- food and meal macro estimation
- places to save or revisit
- product identification, pricing, and order drafting
- civic issue reporting drafts such as 311-style issues
- receipts and expense capture
- document OCR, searchable PDF generation, and filing

Primary runtime outputs:
- `ActionDraft`
- `DecisionRecord`
- `ExecutionReceipt`
- `Report`

Required command surface:
- `look.ingest.image`
- `look.propose.actions`
- `look.execute.action`
- `look.rollback.action`
- `look.status`
- `look.config.set`

## Non-negotiable operating rules
1. Draft-first always.
2. Aggressive preprocessing must happen before clarification.
3. High-risk actions require explicit confirmation per draft.
4. If a permission is missing, drafting still continues and execution-only steps degrade gracefully.
5. Every meaningful decision must emit a `DecisionRecord` with evidence references.
6. Reversible actions must expose rollback information when technically possible.
7. The skill must never require iOS relay upload to function.
8. Device pre-parse from iOS relay is optional evidence, not truth.
9. For event workflows, EXIF capture location must never be treated as the event venue.
10. The package must preserve deterministic storage, schema stability, and auditable logs.

## Package structure
Produce this package structure at minimum:

```text
ocas-look/
  skill.json
  SKILL.md
  references/
    schemas.md
    domain_playbooks.md
    storage_and_config.md
    decision_policy.md
    command_reference.md
```

Optional additions:
- add `assets/` only if it includes genuinely useful example outputs or sample reports
- add `scripts/` only if it includes deterministic helpers that materially improve reliability

Good candidates for `scripts/`:
- package validator
- schema consistency checker
- document filename normalizer
- searchable-PDF generation helper if the skill runtime supports helper scripts

## `skill.json` requirements
Create a valid `skill.json` that includes at minimum:
- `name`: `ocas-look`
- `version`: `1.0.3`
- `author`: `Indigo Karasu`
- `email`: `mx.indigo.karasu@gmail.com`
- a routing-optimized description

The description must make clear that this skill should be used when the user:
- uploads or references an image and wants help determining what action it implies
- wants a flyer, receipt, document, storefront, meal, product, or civic issue image turned into a structured draft
- wants preprocessing and external validation before being asked follow-up questions
- wants decision-ready options instead of raw OCR dumps

The description should include realistic trigger phrasing such as:
- “turn this flyer into a calendar draft”
- “what event is this and where is it?”
- “estimate the macros from this meal photo”
- “add this place to a list”
- “find the best price for this product from the photo”
- “turn this receipt into an expense record”
- “make this document searchable and file it”

The description must not present the skill as:
- a generic OCR layer
- a universal shopping agent
- a generic vision model wrapper
- a broad autonomous executor

## `SKILL.md` requirements
`SKILL.md` must be lean, operational, and highly scannable.
It should contain the runtime behavior and routing logic, not all of the supporting detail.

Target shape:
- roughly 180 to 320 lines
- short sections
- direct operational language
- defer schemas, storage specifics, command details, and domain playbooks to `references/`

`SKILL.md` must include these sections in this order:
1. Title
2. When to use
3. What this skill does
4. Supported domains
5. What this skill does not do
6. Core workflow
7. Clarification policy
8. Confirmation and rollback rules
9. Permission discipline
10. Output contract
11. Support file map
12. Boundaries

## Required support files

### `references/command_reference.md`
This file must define the command interface and the expected inputs and outputs.

Required commands and required behavior:

1. `look.ingest.image`
- Inputs: image(s), EXIF if available, optional `device_preparse`, `ingest_source`
- Outputs: ingest record and `DecisionRecord(ingest)`

2. `look.propose.actions`
- Inputs: optional `ingest_id`, optional user hint
- Outputs: `Report` containing 1 to 3 `ActionDraft`s plus supporting `DecisionRecord`s
- Required decision stages reflected in records: ingest, context, route, research, reduce, draft

3. `look.execute.action`
- Inputs: `draft_id`, explicit confirmation token
- Outputs: `ExecutionReceipt` and `DecisionRecord(execute)`
- Must reject high-risk execution without explicit confirmation

4. `look.rollback.action`
- Inputs: `execution_id`
- Outputs: updated `ExecutionReceipt` and `DecisionRecord(rollback)`
- Must attempt reversal when the underlying action is reversible

5. `look.status`
- Outputs: summary of last ingest, pending drafts, and anything awaiting clarification or confirmation

6. `look.config.set`
- Inputs: config patch
- Outputs: `DecisionRecord(config)`

### `references/storage_and_config.md`
This file must define deterministic storage and configuration behavior.

Required storage rules:
- workspace root: `.look/`
- no writes outside `.look/`
- append-only logs preferred for events and decisions

Required files:
- `.look/config.json`
- `.look/state.json`
- `.look/events.jsonl`
- `.look/decisions.jsonl`
- `.look/reports/`

Recommended directories:
- `.look/artifacts/evidence/`
- `.look/artifacts/drafts/`
- `.look/artifacts/receipts/`

Required config coverage:
- capability toggles by domain
- confidence thresholds
- rate limits
- privacy retention settings
- user profile settings including vegetarian diet
- location and geofence settings
- event evidence weighting and venue-grounding rules
- commerce and ticket preferences
- drive filing rules
- expense ledger routing rules

Include a concrete canonical config example that covers the above and keeps these user-aligned defaults:
- diet: vegetarian
- maps try-list name: `To try`
- event capture-location-as-venue assumption: false
- auto purchase: false
- auto order: false
- drive auto-file default requires confirmation

### `references/schemas.md`
This file must define compact canonical schemas for all key artifacts.

Required shared primitives:
- `DecisionRecord`
- log `Event`
- `Report`

Required skill-specific schemas:
- `ImageEvidence`
- `ContextModel`
- `ActionDraft`
- `ExecutionReceipt`

Schema requirements:
- required fields
- optional fields
- field meanings
- risk-relevant constraints
- enough detail for stable machine-readable use

The schemas must preserve the original functional surface, including these concepts:
- `ingest_source`
- optional `device_preparse`
- EXIF object with GPS and timezone data
- extracted entities such as dates, times, addresses, phones, URLs, place names, and product strings
- ranked contexts with confidence
- action draft type, risk, confidence, summary, fields, evidence refs, clarifications, next step
- execution receipt external references for calendar, maps, orders, reports, health entries, and drive files

### `references/decision_policy.md`
This file must define the safety model, risk tiers, confirmation rules, rollback expectations, permission degradation behavior, and no-hallucination policy.

It must explicitly cover:
- low / medium / high risk tiers
- high-risk actions: purchases, 311 submission, health writes, drive writes or moves
- medium-risk actions: calendar creation and maps writes
- low-risk actions: pricing comparisons and candidate lists
- missing-permission behavior
- rate limiting
- requirement to log safety blocks as `DecisionRecord(safety.rate_limit)` or equivalent
- never invent OCR, barcodes, prices, ticket inventory, or license plates

### `references/domain_playbooks.md`
This file must preserve the detailed domain implementations. Do not flatten these into vague summaries.

It must contain one section for each domain below.

#### Events
Preserve these rules:
- EXIF capture location and city are weak signals only
- event venue and timezone must be grounded through strong signals
- strong grounding requires at least one of:
  - QR or URL resolving to canonical event page with venue and datetime
  - full street address in flyer text that geocodes cleanly
  - venue name plus credible official venue confirmation
- city text may constrain search but cannot decide venue or timezone alone

Required preprocessing:
- title or artist extraction
- date and time parsing
- doors versus show time interpretation where possible
- venue name extraction
- city extraction
- URL and QR extraction
- canonical page fetch when URL or QR exists
- available dates and times retrieval where possible
- pricing, fees, and inventory cues where available
- venue address resolution
- timezone determination from grounded venue
- calendar conflict checks only after venue and timezone are grounded
- option ranking: conflict-free first, then price and availability

Required draft types:
- `calendar_hold`
- `calendar_event`
- `ticket_purchase`

Required event clarification style:
- only after option reduction
- choose among conflict-free options where possible
- ask for ticket quantity if required
- ask buy versus hold versus calendar-only only when needed

#### Food macros
Preserve these rules:
- use context and nearby location to identify restaurant when possible
- fetch menu when possible
- apply vegetarian constraint before asking clarification questions
- use image cues plus cuisine and menu constraints to narrow candidates
- enrich macros using authoritative restaurant nutrition when available, otherwise best available sources
- infer portion size and likely consumption amount
- produce macro range plus confidence

Required clarification behavior:
- ask only if more than one viable vegetarian candidate remains
- ask about consumption state only after reduction, for example all / some / shared / leftovers

Required draft type:
- `health_macros`

#### Places
Preserve these rules:
- OCR signage
- use EXIF GPS and nearby POIs to resolve candidate places
- dedupe against target list
- only clarify if ambiguity remains between a small number of adjacent candidates

Required draft type:
- `maps_to_try`

#### Products
Preserve these rules:
- barcode dominates when present
- otherwise use OCR brand or model plus image cues to resolve exact variant
- eliminate near matches
- fetch offers from preferred retailers
- compare availability and delivery estimates
- reduce to the cheapest versus fastest options unless the same option wins both

Required draft types:
- `product_pricing`
- `product_order`

Clarification rules:
- only for unresolved variant, quantity, or retailer choice

#### Civic issues
Preserve these rules:
- determine jurisdiction from GPS
- validate driveway or home geofence if configured
- if geofence is not configured, keep as draft-only and require explicit location confirmation
- description must stay factual and must not infer identity or blame

Required draft type:
- `civic_report_311`

#### Receipts
Preserve these rules:
- receipt classifier using layout plus OCR plus currency patterns
- extract merchant, date, subtotal, tax, tip, total
- resolve merchant using nearby POIs if OCR is weak
- categorize via merchant type and line items if present
- route to ledger or project using config
- never blend ledgers when config separates them

Required draft type:
- `expense_entry`
- optional paired `drive_file_document` draft when filing is also proposed

Clarification rules:
- only for ledger, project, or business-versus-personal routing when ambiguous

#### Documents
Preserve these rules:
- document classifier
- OCR plus deskew
- generate searchable PDF with image plus text layer
- infer document type such as invoice, letter, insurance, legal, medical, or general
- propose drive folder plus filename format `[project]_[topic]_[yyyymmdd]`

Required draft type:
- `drive_file_document`

Clarification rules:
- only for low-confidence folder disambiguation, ideally no more than two options

## Core execution workflow
The skill must implement this 8-stage pipeline per ingest and reflect it in the runtime instructions:
1. Ingest
2. Extraction merge
3. Context inference
4. Domain routing
5. External research
6. Constraint filtering
7. Option reduction
8. Draft generation

Detailed rules:
- if `device_preparse` exists, merge it as evidence only
- if `device_preparse` does not exist, local extraction still runs
- clarification is allowed only after option reduction
- drafts must be decision-ready, not raw extraction dumps

## iOS relay handling
The built skill must explicitly preserve this behavior:
- device pre-parse through Apple Intelligence or VisionKit is optional
- it is only available when images arrive via the iOS relay app
- uploads from Messages, Telegram, Email, or web upload must still fully work
- the skill must never ask the user to re-upload through iOS relay as a prerequisite
- when relay pre-parse exists, the skill must still run validation and merge logic instead of trusting it blindly

## Permission discipline
The built skill must preserve this operating model:
- default deny, request minimally
- drafting continues even when execution permissions are missing
- blocked execution steps are reported as blocked, not silently skipped
- permission grants and denials are logged

Expected permission groups should cover these categories where supported by the target ecosystem:
- photo and image read access
- EXIF read access
- optional location access
- optional calendar read and write
- optional maps read and write
- optional commerce read and write
- optional civic reporting write
- optional health write
- optional drive read and write
- optional network access for validation and research

## Acceptance criteria
The final package must satisfy all of the following.

1. Build correctness
- package installs as an Agent Skill
- metadata is complete and coherent
- routing description is strong and realistic
- support files are referenced clearly from `SKILL.md`

2. Functional completeness
- all seven domains above are represented
- all six required commands exist
- all four output artifact types are defined
- rollback exists for reversible actions

3. Draft-first behavior
- propose step yields 1 to 3 `ActionDraft`s or a constrained clarification state
- no high-risk action executes without explicit confirmation token

4. Event safety
- if venue and timezone are not strongly grounded, the skill uses `calendar_hold` rather than `calendar_event`
- city text can constrain search but cannot decide venue and timezone alone
- conflict checks are not presented as definitive until grounding is complete

5. iOS relay optionality
- given non-relay ingest with no pre-parse, the skill still drafts successfully using local extraction
- given relay ingest with pre-parse, the skill merges it as evidence and still validates
- the skill never requires relay upload for functionality

6. Food constraint handling
- if restaurant and menu are resolved, vegetarian filtering happens before clarification

7. Reversibility
- calendar and maps actions expose rollback references
- rollback command attempts undo where supported

8. Persistence
- state and logs persist across restarts or repeated invocations

9. Permission degradation
- missing execution permissions do not block draft generation
- blocked execution steps are surfaced explicitly

10. No silent scope loss
- the final built skill preserves the functionality present in the original Look concept rather than compressing it into a generic image utility

## Responsibility Boundary

Look converts user-provided images into validated, decision-ready action drafts.

Look does not perform general web research. Topic and fact research belong to Sift.

Look does not perform OSINT investigations on people. Person-focused investigations belong to Scout.

---

## Optional Skill Cooperation

This skill may cooperate with other skills when present but must never depend on them.
If a cooperating skill is absent, this skill must still function normally.

- Sift — delegate web research for event validation, product pricing, and restaurant identification.
- Weave — query social graph for person identification in images.
- Taste — apply preference signals for food and product recommendations.
- Voyage — provide travel context for location-based image interpretation.
- Relay — receive device-originated images and pre-parse data from iOS clients.

---

## Journal Outputs

This skill emits the following journal types as defined in the OCAS Journal Specification (spec-ocas-Journals.md):

- Observation Journal

Look emits Observation Journal entries recording image ingestion evidence, context inferences, and domain routing decisions.

---

## Visibility

visibility: public

---

## Universal OKRs

This skill must implement the universal OKRs defined in the OCAS Journal Specification (spec-ocas-Journals.md).

Required universal OKRs:

- Reliability: success_rate >= 0.95, retry_rate <= 0.10
- Validation Integrity: validation_failure_rate <= 0.05
- Efficiency: latency trending downward, repair_events <= 0.05
- Context Stability: context_utilization <= 0.70
- Observability: journal_completeness = 1.0

Skill-specific OKRs should be defined in the built SKILL.md to measure domain-relevant outcomes.

---

## Output format required from the coder
Return:
1. the package tree
2. full contents of every created file
3. a short validation checklist confirming the acceptance criteria above

Do not return a build memo instead of the package.
