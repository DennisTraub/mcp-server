---
title: "fix: Address P1/P2 security review findings for HTTP fetcher"
type: fix
status: completed
date: 2026-03-12
origin: Code review of commit 2774e48 on fix/p2-security-hardening
---

# fix: Address P1/P2 security review findings for HTTP fetcher

## Problem

Code review of commit `2774e48` found security gaps in `doc_fetcher.py`:

1. **`_get()` has no URL validation** — `_is_valid_doc_url()` in `server.py` only gates `fetch_doc`. The `search_docs` and `parse_llms_txt` paths reach `_get()` ungated. A `file:///etc/passwd` URL bypasses the redirect handler entirely (it only fires on HTTP 3xx).
2. **Redirect handler leaks hostnames and permits non-HTTPS targets** — error message reveals `target.hostname` (e.g., `169.254.169.254`), and the condition only checks same-scheme instead of enforcing HTTPS-only.
3. **`ValueError` for oversized responses** — semantically wrong; callers catching `ValueError` broadly could swallow this security boundary.
4. **No Content-Length pre-check** — reads up to 2MB before rejecting obviously oversized responses.

## Changes

All in `src/strands_mcp_server/utils/doc_fetcher.py`. Two concerns separated: `_get()` enforces transport safety (HTTPS, no userinfo), `server.py` keeps the hostname allowlist as defense-in-depth.

### 1. Add `_validate_fetch_url()` called at the top of `_get()`

```python
def _validate_fetch_url(url: str) -> None:
    parsed = urllib.parse.urlparse(url)
    if parsed.scheme != "https":
        raise urllib.error.URLError(f"only https URLs are allowed, got {parsed.scheme!r}")
    if "@" in (parsed.netloc or ""):
        raise urllib.error.URLError("URLs with userinfo (@) are not allowed")
    if not parsed.hostname:
        raise urllib.error.URLError("URL has no hostname")
```

Rejects non-HTTPS schemes, `@`-userinfo bypasses (e.g., `https://good.com@evil.com/`), and empty hostnames. Every call path through `_get()` is now gated.

### 2. Fix redirect handler (2-line change)

Change condition from `target.scheme != original.scheme` to `target.scheme != "https"` — explicitly allowlists HTTPS-only.

Remove hostnames from error message: `f"cross-host redirect blocked ({code})"` — no attacker-controlled data in exceptions.

### 3. `ValueError` → `RuntimeError` for oversized responses

One-line change. Oversized response is a runtime condition, not a caller error.

### 4. Content-Length pre-check

```python
with _opener.open(req, timeout=doc_config.timeout) as r:
    content_length = r.headers.get("Content-Length")
    if content_length and content_length.isdigit() and int(content_length) > doc_config.max_response_bytes:
        raise RuntimeError(f"response exceeds {doc_config.max_response_bytes} bytes")
    data = r.read(doc_config.max_response_bytes + 1)
    ...
```

Advisory fast-path only — `Content-Length` can be spoofed, so the `read(N+1)` check remains as the authoritative guard. Uses `isdigit()` to safely ignore malformed header values.

## Tests

In `tests/test_doc_fetcher.py`:

**Fix 2 existing tests:**
- `test_error_does_not_leak_target_url` — assert `"169.254.169.254" not in str(exc_info.value)` (current assertion only checks the full URL, not the hostname — false assurance)
- `test_oversized_response_raises_value_error` — change to `pytest.raises(RuntimeError, ...)`

**Add 4 new tests:**
- `test_non_https_url_rejected` — parametrized with `["http://x.com/", "ftp://x.com/", "file:///etc/passwd"]`
- `test_userinfo_url_rejected` — `_get("https://a@evil.com/")`
- `test_https_url_passes_validation` — happy path with mocked opener
- `test_content_length_precheck_rejects_oversized` — mocked Content-Length header

## Acceptance Criteria

- [x] `_get()` rejects non-HTTPS URLs and `@`-userinfo URLs with `URLError`
- [x] Redirect handler only allows HTTPS targets, error message has no hostnames
- [x] Oversized response raises `RuntimeError`
- [x] Content-Length pre-check rejects declared-oversized responses
- [x] All existing tests pass, `ruff check` clean

## Affected Files

| File | Changes |
|------|---------|
| `src/strands_mcp_server/utils/doc_fetcher.py` | `_validate_fetch_url`, redirect handler fix, exception type, Content-Length pre-check |
| `tests/test_doc_fetcher.py` | Fix 2 existing tests, add 4 new tests |

## Sources

- **Code review:** Commit `2774e48` on `fix/p2-security-hardening`
- **Prior plan:** `docs/plans/2026-03-09-refactor-p2-security-hardening-and-code-quality-plan.md`
- **Existing validation:** `src/strands_mcp_server/server.py:11-17` (`_is_valid_doc_url`)
