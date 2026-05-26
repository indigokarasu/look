# Look — Core Workflow

## Procedure

1. Ingest image(s) with EXIF if available
2. Merge extraction evidence (OCR, entities, context)
3. Infer context and likely intent
4. Route to appropriate domain
5. Research and validate externally
6. Filter by constraints (dietary, preferences, permissions)
7. Reduce options before asking questions
8. Generate 1–3 decision-ready ActionDrafts
9. Emit Signal files for extracted entities (places, events, products). Signal schema: see `references/schemas.md`.
10. Write journal via `look.journal`

Clarification happens only after option reduction, not before.

## Error Handling

| Failure | Detection | Response |
|---------|-----------|----------|
| Image unreadable / corrupt | `vision_analyze` returns error or empty | Report to user, request re-upload |
| Domain routing ambiguous | Low confidence across all domains | Present top 2 domain options, ask user |
| Reverse search returns empty | No results from Yandex or Google | Note "no external validation found" in draft, proceed with extracted data |
| Draft generation fails | Missing required fields (date, amount, etc.) | Mark draft as incomplete, list missing fields, ask user |
| Execution blocked (permissions) | Tool returns auth/permission error | Report blocked execution, save draft for manual action |
| API rate limit (Imgur, Yandex) | HTTP 429 or similar | Wait 30s, retry once. If still failing, report and continue without reverse search |
