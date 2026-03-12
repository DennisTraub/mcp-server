---
title: "SSRF and DoS hardening for doc_fetcher HTTP client"
date: 2026-03-12
category: security-issues
tags: [ssrf, dos, http-redirect, url-validation, response-size, mcp-server]
severity: high
component: strands_mcp_server/utils/doc_fetcher.py
symptom: "_get() followed redirects to arbitrary hosts, accepted non-HTTPS schemes, and read unbounded responses"
root_cause: "urllib.request.urlopen follows redirects by default with no host/scheme checks; no URL validation at the fetch layer; no response size limit"
commit: 976aa8b
branch: fix/p2-security-hardening
prior_commit: 5331077
---

# SSRF and DoS hardening for doc_fetcher HTTP client

## Problem

The `_get()` function in `doc_fetcher.py` fetches documentation URLs for the MCP server's `fetch_doc` and `search_docs` tools. It used `urllib.request.urlopen()` directly with no security controls, exposing multiple attack vectors:

1. **SSRF via scheme abuse** — No validation that URLs use HTTPS. An attacker could supply `file:///etc/passwd`, `http://169.254.169.254/` (cloud metadata), or `ftp://` URLs. While `server.py` had `_is_valid_doc_url()` gating `fetch_doc`, other code paths (`search_docs`, `parse_llms_txt`) called `_get()` ungated.

2. **SSRF via open redirects** — `urlopen` follows HTTP 3xx redirects by default. A legitimate HTTPS URL could redirect to an internal host (e.g., AWS metadata endpoint), bypassing any URL validation on the initial request.

3. **DoS via unbounded response** — `r.read()` consumed entire responses with no size limit. A malicious or unexpectedly large page could exhaust process memory.

4. **Credential leakage** — URLs with userinfo (`https://user:pass@host/`) were accepted, enabling credential smuggling via URL authority.

## Root Cause

`urllib.request.urlopen` is designed for general-purpose HTTP fetching and follows redirects transparently. In a tool-use context (MCP server accepting URLs from LLM-generated tool calls), this default behavior creates SSRF risk because:

- URL validation only at the entry point (`server.py`) leaves internal code paths unprotected
- Redirect targets bypass any entry-point validation
- No defense-in-depth at the transport layer

## Solution

The fix was implemented across two iterations: an initial hardening (plan from 2026-03-09) followed by a security review that caught additional gaps (plan from 2026-03-12). The final commit consolidates all fixes.

### 1. URL validation at the fetch layer

Added `_validate_fetch_url()` called at the top of `_get()`, gating every code path:

```python
def _validate_fetch_url(url: str) -> None:
    parsed = urllib.parse.urlparse(url)
    if parsed.scheme != "https":
        raise urllib.error.URLError("only https URLs are allowed")
    if "@" in (parsed.netloc or ""):
        raise urllib.error.URLError("URLs with userinfo (@) are not allowed")
    if not parsed.hostname:
        raise urllib.error.URLError("URL has no hostname")
```

### 2. Same-host redirect handler

Replaced default redirect behavior with `_SameHostRedirectHandler` that validates redirect targets with the same checks as initial requests, plus a same-host constraint:

```python
class _SameHostRedirectHandler(urllib.request.HTTPRedirectHandler):
    def redirect_request(self, req, fp, code, msg, headers, newurl):
        _validate_fetch_url(newurl)
        original = urllib.parse.urlparse(req.full_url)
        target = urllib.parse.urlparse(newurl)
        if target.hostname != original.hostname:
            raise urllib.error.URLError(f"cross-host redirect blocked ({code})")
        return super().redirect_request(req, fp, code, msg, headers, newurl)

_opener = urllib.request.build_opener(_SameHostRedirectHandler)
```

Key design decisions:
- **Same-host, not no-redirect** — allows legitimate path redirects (e.g., trailing slash normalization) while blocking cross-host SSRF
- **Validates redirect targets** — applies HTTPS-only, no-userinfo checks to redirect URLs too
- **Opaque error messages** — error text includes the HTTP status code but not attacker-controlled hostnames

### 3. Bounded response read

```python
with _opener.open(req, timeout=doc_config.timeout) as r:
    data = r.read(doc_config.max_response_bytes + 1)
    if len(data) > doc_config.max_response_bytes:
        raise RuntimeError(f"response exceeds {doc_config.max_response_bytes} bytes")
    return data.decode("utf-8", errors="ignore")
```

Uses `read(N+1)` pattern: reads one byte past the limit to detect oversized responses without buffering the entire body.

### 4. Configurable size limit

```python
# config.py
max_response_bytes: int = 2 * 1024 * 1024  # 2 MB default
```

## Verification

13 unit tests added in `tests/test_doc_fetcher.py` across three test classes:

| Class | Tests | Coverage |
|-------|-------|----------|
| `TestSameHostRedirectHandler` | 3 | Same-host allowed, scheme downgrade blocked, cross-host blocked |
| `TestResponseSizeCap` | 3 | Oversized rejected, boundary (exact limit) succeeds, normal succeeds |
| `TestUrlValidation` | 7 | Non-HTTPS rejected (parametrized: http/ftp/file), userinfo rejected, no hostname rejected, `_get()` rejects non-HTTPS, valid HTTPS passes |

## Evolution

This fix went through two iterations:

1. **Initial plan (2026-03-09)**: Blocked all redirects entirely (`_NoRedirectHandler`) and added size cap. Raised `ValueError` for oversized responses.

2. **Security review (2026-03-12)**: Found gaps — `_get()` had no URL validation (only `server.py` did), error messages leaked hostnames, `ValueError` was semantically wrong. Evolved to same-host redirect policy, added `_validate_fetch_url()`, changed to `RuntimeError`.

## Prevention Strategies

### For this codebase
- **Validate at the transport layer, not just the entry point** — any function that makes HTTP requests should enforce its own URL policy, independent of callers
- **Defense in depth** — `server.py` allowlist + `doc_fetcher.py` transport checks operate as independent layers

### For MCP servers generally
- **Treat all URLs as untrusted input** — LLM-generated tool calls may contain any URL, including internal addresses
- **HTTPS-only by default** — there is rarely a legitimate reason for an MCP documentation tool to fetch HTTP, FTP, or file URLs
- **Bound all reads** — always use `read(max_size)`, never unbounded `read()`
- **Restrict redirects** — at minimum, validate redirect targets; prefer same-host-only or no-redirect policies
- **Keep error messages opaque** — never include attacker-controlled data (hostnames, URLs) in error messages that might be surfaced to users

### Remaining considerations
- **DNS rebinding** — `_validate_fetch_url` checks the URL string but not the resolved IP. A hostname could resolve to `127.0.0.1` or `169.254.169.254`. For higher-security contexts, consider resolving DNS first and checking against a private IP blocklist.
- **Content-Length pre-check** — the current implementation reads up to 2MB before rejecting. An advisory `Content-Length` header check could reject obviously oversized responses without reading (added in the review plan but simplified out of the final commit).

## Related

- **Prior SSRF fix**: Commit `5331077` — "fix: add url validation to mitigate SSRF issue (#29)" — added `_is_valid_doc_url()` in `server.py`
- **Initial plan**: `docs/plans/2026-03-09-refactor-p2-security-hardening-and-code-quality-plan.md`
- **Review findings plan**: `docs/plans/2026-03-12-fix-p2-security-review-findings-plan.md`
- **Existing solution doc**: `docs/solutions/performance-issues/mcp-fetch-doc-smart-sectioning-token-reduction.md` (same component, different concern)
