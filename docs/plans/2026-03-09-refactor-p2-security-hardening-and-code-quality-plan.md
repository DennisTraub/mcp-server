---
title: P2 security hardening
type: fix
status: completed
date: 2026-03-09
origin: Code review findings from feat/browse-doc branch review (2026-03-09)
---

# fix: P2 security hardening

## Overview

Address two security hardening findings from the code review of the `feat/browse-doc` branch: block HTTP redirects in the fetch layer and cap response size to prevent memory exhaustion.

## Problem Statement

1. **Redirect-following SSRF**: `_get()` in `doc_fetcher.py` uses `urllib.request.urlopen` which follows HTTP redirects by default. URL validation in `_is_valid_doc_url` only checks the initial URL. An open redirect on `strandsagents.com` could bypass SSRF protection and reach internal endpoints (e.g., cloud metadata at `169.254.169.254`).

2. **Unbounded response size**: `r.read()` in `doc_fetcher.py:46` reads the entire HTTP response with no size limit. Combined with the unbounded `_URL_CACHE` dict in `cache.py`, memory grows monotonically. A large page (or many pages) could exhaust process memory.

## Proposed Solution

Single commit targeting `doc_fetcher.py` and `config.py` only.

### 1. Block redirects entirely

The server only fetches curated docs from `strandsagents.com`. Legitimate doc pages don't redirect. Blocking all redirects is the simplest and most secure approach.

```python
# src/strands_mcp_server/utils/doc_fetcher.py

class _NoRedirectHandler(urllib.request.HTTPRedirectHandler):
    def redirect_request(self, req, fp, code, msg, headers, newurl):
        raise urllib.error.URLError(f"redirects not followed: {code} -> {newurl}")

_opener = urllib.request.build_opener(_NoRedirectHandler)
```

Update `_get()` to use `_opener.open(req, timeout=...)` instead of `urllib.request.urlopen(req, timeout=...)`.

### 2. Cap response size

Add `max_response_bytes` to `config.py` and enforce in `_get()`:

```python
# config.py
max_response_bytes: int = 2 * 1024 * 1024  # 2 MB

# doc_fetcher.py _get()
data = r.read(doc_config.max_response_bytes + 1)
if len(data) > doc_config.max_response_bytes:
    raise ValueError(f"response exceeds {doc_config.max_response_bytes} bytes")
return data.decode("utf-8", errors="ignore")
```

### Tests (new file: `tests/test_doc_fetcher.py`)

- [x] Redirect returns raise `URLError` (mock `urlopen` to return 302)
- [x] Oversized response raises `ValueError`
- [x] Normal response within size limit works unchanged

## Blast Radius

| File | What changes |
|------|-------------|
| `src/strands_mcp_server/utils/doc_fetcher.py` | `_NoRedirectHandler` class, `_opener`, update `_get()` to use opener + size cap |
| `src/strands_mcp_server/config.py` | Add `max_response_bytes: int = 2 * 1024 * 1024` |
| `tests/test_doc_fetcher.py` (new) | 3 unit tests for redirect blocking and size limiting |

## Acceptance Criteria

- [x] HTTP redirects raise `URLError` in `doc_fetcher._get()`
- [x] Response size capped at 2 MB (configurable via `config.py`)
- [x] 3 new unit tests pass for redirect blocking and size limiting
- [x] All existing 46 tests still pass (50 total now)
- [x] `ruff check` and `ruff format` pass
- [x] No behavioral changes for normal (non-redirect, <2MB) fetches

## Deferred (separate follow-up PR)

These code quality P2 items were scoped out to keep this change focused:

- **Explicit section dict construction in `server.py:185`** — replace `startswith("_")` filter with explicit key list
- **Extract `_first_prose_paragraph()` helper** — deduplicate ~20 LOC between `make_snippet` and `make_section_summary`

## Risks

| Risk | Mitigation |
|------|------------|
| Legitimate redirect on strandsagents.com breaks fetching | Unlikely for doc pages; if needed, switch to re-validating handler later |
| 2 MB limit too low for some pages | Configurable via `config.py`; can adjust without code changes |

## Sources

- **Code review:** `feat/browse-doc` branch review (2026-03-09) — security-sentinel findings
- **SSRF context:** `urllib.request.urlopen` follows 301/302/303/307/308 redirects by default
- **Files affected:**
  - `src/strands_mcp_server/utils/doc_fetcher.py:32-46`
  - `src/strands_mcp_server/config.py:14-18`
