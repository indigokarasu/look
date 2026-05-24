# 👁️ Look

> **Image-to-action — takes a photo and produces a validated action draft.**

## Why Look?

You take a photo of a receipt, a whiteboard, a menu, a document — and then what? Look bridges the physical and digital by taking a user-provided image, inferring what you probably want done with it, and producing a validated action draft across domains: calendar events, meal macros, places, product comparisons, receipts, documents, and civic reports.

Skill packages follow the [agentskills.io](https://agentskills.io/specification) open standard and are compatible with OpenClaw, Hermes Agent, Claude, and any agentskills.io-compliant client.

## Quick Start

```
# Send an image
"Here's a photo of this receipt — what can you do with it?"

# Whiteboard capture
"Parse this whiteboard photo into action items"

# Menu scan
"What are the good options on this menu?"
```

## What It Does

Look takes a user-provided image, infers intent, and produces a validated action draft. It resolves ambiguity through research and option reduction before asking clarifying questions. Nothing executes without explicit per-draft confirmation. It handles receipts, documents, menus, whiteboards, products, and civic reports.

## Dependencies

- Vision API (image analysis)
- [Sift](https://github.com/indigokarasu/sift) — research for disambiguation

---

*Look is part of the [OCAS Agent Suite](https://github.com/indigokarasu).*