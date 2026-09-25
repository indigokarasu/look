# Python Package Installation for OCAS Skills

## The Venv Problem

Hermes Agent runs inside a virtual environment (locate it: `python3 -c "import sys; print(sys.prefix)"`; examples below use `<hermes-venv>` as the venv path).
System `pip` may install to `/usr/local/lib/python3.X/dist-packages/` instead of the venv's `site-packages/`, causing `ModuleNotFoundError` at runtime even though `pip` reported success.

## Correct Pattern

Always install into the **venv**, not system packages:

```bash
# Correct — use the venv's pip
<hermes-venv>/bin/pip3 install <package>

# Verify import works from the venv's Python
<hermes-venv>/bin/python -c "import <module>; print('OK')"
```

Do NOT rely on bare `pip install` or `pip3 install` — they may target system dist-packages.

## pip/requests Version Conflict

Installing a package may downgrade `requests` (e.g., `google-image-source-search` pulls `requests==2.31.0`). Hermes Agent requires `requests==2.33.0`. After installing any package, check and restore:

```bash
<hermes-venv>/bin/pip3 install requests==2.33.0
```

The order is: install the skill package first, then pin requests back.

## google-image-source-search Setup (ocas-look)

Full setup for the reverse image search fallback used by ocas-look:

```bash
# Install the package
<hermes-venv>/bin/pip3 install google-image-source-search

# Restore pinned requests version
<hermes-venv>/bin/pip3 install requests==2.33.0

# Verify
<hermes-venv>/bin/python -c "from google_img_source_search import ReverseImageSearcher; print('OK')"
```

### Usage

```python
from google_img_source_search import ReverseImageSearcher

searcher = ReverseImageSearcher()

# Search by publicly accessible image URL
results = searcher.search("https://example.com/photo.jpg")

# Search by local file (uploads directly)
results = searcher.search_by_file("/path/to/photo.png")

for r in results:
    print(f"Title: {r.page_title}")
    print(f"Page:  {r.page_url}")
    print(f"Image: {r.image_url}")
```

Each result contains `page_title`, `page_url`, and `image_url`. Results are ordered by Google's relevance ranking.

**⚠️ NOTE:** Google blocks image search uploads from cloud/server IPs. This will not work from VPS/cloud environments — use Yandex reverse image search instead (see SKILL.md).

## Verification Checklist

After installing any pip package for a skill:

1. `__venv__/bin/python -c "import MODULE"` succeeds
2. `requests` version unchanged: `__venv__/bin/pip3 show requests | grep Version`
3. The package appears in `__venv__/bin/pip3 list`
