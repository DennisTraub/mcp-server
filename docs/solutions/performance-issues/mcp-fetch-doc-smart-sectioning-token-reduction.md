---
title: "fetch_doc tool redesigned with smart sectioning to reduce token usage"
date: 2026-03-10
category: performance-issues
component: strands_mcp_server/server.py
tags:
  - mcp
  - token-optimization
  - smart-sectioning
  - fetch_doc
  - ssrf-hardening
  - backward-compatibility
severity: medium
symptoms:
  - "fetch_doc returns entire documentation pages consuming 30-50KB per call"
  - "excessive token usage in LLM workflows for large API reference docs"
  - "no way to request a specific section of a document"
  - "URL validation used naive startswith check vulnerable to SSRF"
root_cause: "fetch_doc lacked sectioning support, always returning full page content regardless of need"
resolution: "Added TOC/section/small-doc modes to fetch_doc, reducing token usage by 80-85% with urlparse-based SSRF hardening"
pr: "https://github.com/strands-agents/mcp-server/pull/32"
commit: "0b7ea7c"
---

# fetch_doc Smart Sectioning -- 80-85% Token Reduction

## Problem

The `fetch_doc` MCP tool returned entire documentation pages in a single response, consuming excessive tokens in LLM conversations. Large documents such as API references (30-50KB) were sent in full even when users only needed a specific section, leading to unnecessary cost and context window pressure. There was no mechanism for an LLM to inspect document structure before requesting content.

## Investigation Steps

1. PR #32 introduced a new tool initially named `browse_doc` with smart sectioning to replace the monolithic `fetch_doc`.
2. Reviewer (@mkmeral) flagged that renaming the tool would break existing `allowedTools` and `autoApprove` configurations in downstream IDE setups, requesting a backward-compatible rename back to `fetch_doc`.
3. SSRF URL validation was upgraded from naive `str.startswith()` checks to `urlparse`-based validation to block subdomain tricks, userinfo bypass, and scheme downgrade attacks.
4. A `preamble` field was added per reviewer feedback so TOC mode returns introductory content between the H1 title and the first H2 section.
5. Integration tests were trimmed to avoid fragile external dependencies on live documentation content.
6. Test classes were renamed to match the restored `fetch_doc` naming convention.
7. The `uri` parameter was made optional (defaulting to `""`) so LLMs can call `fetch_doc()` with no arguments to receive a catalog of all available URLs.
8. Final formatting pass applied via `hatch fmt`.

## Root Cause

The original `fetch_doc` implementation had no content-aware chunking strategy. It fetched the raw markdown for a URL and returned it verbatim, regardless of document size. There was no mechanism for an LLM to inspect document structure before requesting content, so every call paid the full token cost of the entire page.

## Solution

The tool was redesigned with three operating modes within the same `fetch_doc` endpoint, controlled by the presence or absence of the `section` parameter.

**Step 1 -- Small doc auto-detection:** Documents under `SMALL_DOC_THRESHOLD` (8192 bytes) are returned in full immediately, since sectioning overhead would exceed the savings. Documents with no parseable H2 sections are also returned in full. Both cases include a `reason` field (`"size"` or `"no_sections"`).

**Step 2 -- TOC mode (no `section` param):** The tool calls `parse_sections()` to build a hierarchical section tree from ATX headers, then returns section IDs, titles, summaries, and children. It also calls `extract_preamble()` to include content between the H1 heading and the first H2. Internal fields (prefixed with `_`) are stripped before returning.

**Step 3 -- Section mode (`section="3"` or `"3.2"`):** The tool calls `extract_section()` to return the full markdown of a single section by dotted ID. It supports two levels of nesting: H2 sections and H3 children. Precise slicing uses stored `_start` character offsets and next-sibling detection to find the end boundary.

**Step 4 -- Security hardening:** URL validation was moved from string prefix matching to `urlparse`-based checks, ensuring only `https` scheme and `strandsagents.com` hostname are permitted.

### Key Code Changes

**`src/strands_mcp_server/server.py`**
- `fetch_doc()` accepts optional `uri` (default `""`) and `section` (default `""`) parameters.
- Three-way branching: small doc detection, section extraction, and TOC mode (default).
- `_is_valid_doc_url()` uses `urlparse` to validate scheme and hostname.
- Internal `_`-prefixed keys stripped from section dicts before returning TOC results.

**`src/strands_mcp_server/utils/text_processor.py`**
- `parse_sections(content)` -- Scans ATX headers via regex, skips headers inside fenced code blocks using `_code_fence_ranges()` / `_in_code_block()`, builds flat list of H2 sections each with a `children` list (H3+). Stores `_start` offsets and `_children_internal` for extraction.
- `extract_section(content, section_id, sections)` -- Resolves dotted IDs to character offset ranges using `_start` values and next-sibling boundary detection. Returns `None` for invalid IDs or 3+ nesting levels.
- `extract_preamble(content)` -- Finds the first real H2 header (not inside code fences), returns everything before it with the H1 line stripped.
- `make_section_summary(section_text)` -- Extracts first meaningful prose paragraph, falls back to `"Contains: child1, child2, ..."`.
- `_truncate(text, max_chars)` -- Truncates with Unicode ellipsis character.

## Result

80-85% token reduction for typical documentation workflows. Users now follow a three-step pattern:

1. `search_docs("your query")` -- find relevant URLs
2. `fetch_doc(uri="...")` -- browse the table of contents with summaries
3. `fetch_doc(uri="...", section="3")` -- read only the section needed

## Prevention Strategies

**Default to minimal payloads.** Any MCP tool that returns document content should never send the full body by default. Design the tool contract so the first call returns metadata (titles, summaries, structure) and a follow-up call fetches the specific slice.

**Set hard byte budgets early.** Define a constant like `SMALL_DOC_THRESHOLD` from day one. Having a named threshold forces the conversation about "how big is too big?" during design rather than after a token bill surprise.

**Treat token cost as a first-class acceptance criterion.** Add a CI check or integration test that asserts the response size of TOC mode stays under a defined ceiling (e.g., 2KB for a 50KB doc).

**Audit every return path.** This tool has four return shapes: catalog, small-doc, TOC, and section. Each path must be reviewed independently for payload size. Fallback paths (like `no_sections` returning full content) can silently bypass the efficiency mechanism.

## Best Practices

**Code-fence-aware parsing is non-negotiable.** Any markdown parser that splits on headers must track fenced code block ranges and skip headers inside them. The `_code_fence_ranges` / `_in_code_block` pattern is clean and reusable.

**Use dotted hierarchical IDs, not heading text.** Section IDs like `"3.2"` are stable, unambiguous, and short. Heading text is fragile (duplicates, special characters, localization).

**Validate URLs with `urlparse`, not string matching.** Checking `parsed.hostname` prevents bypass vectors like `strandsagents.com.evil.com` and `strandsagents.com@evil.com`.

**Strip internal fields before returning.** Use an underscore-prefix convention for internal-only data in section dicts, filtered with `not k.startswith("_")`.

## Test Coverage Notes

**Well covered (37+ tests):**
- Section parsing: hierarchy, IDs, children, code-fence immunity, empty content
- Section extraction: top-level, child, last section, invalid IDs (parametrized)
- Preamble extraction: H1 stripping, empty preamble, no H2, code-block immunity
- Summary generation: paragraph extraction, truncation, code-block skipping
- Server-level: TOC mode, small doc bypass, section mode, SSRF vectors, URL catalog

**Gaps worth noting:**
- No test for 3+ nesting depth (code returns `None` but no test asserts it)
- No test for extremely large preambles (returned untruncated, could negate savings)
- No performance regression test asserting the 80-85% reduction claim
- No test for non-UTF-8 or CJK-heavy docs where byte vs character count diverges

## Cross-References

- **Brainstorm:** [docs/brainstorms/2026-03-09-rename-browse-doc-to-fetch-doc-brainstorm.md](../../brainstorms/2026-03-09-rename-browse-doc-to-fetch-doc-brainstorm.md) -- tool naming and backward compatibility decisions
- **Rename plan:** [docs/plans/2026-03-09-refactor-rename-browse-doc-to-fetch-doc-plan.md](../../plans/2026-03-09-refactor-rename-browse-doc-to-fetch-doc-plan.md) -- blast radius analysis (32 occurrences across 5 files)
- **Security plan:** [docs/plans/2026-03-09-refactor-p2-security-hardening-and-code-quality-plan.md](../../plans/2026-03-09-refactor-p2-security-hardening-and-code-quality-plan.md) -- redirect blocking and response size capping (deferred items)
- **PR:** [#32](https://github.com/strands-agents/mcp-server/pull/32)
- **Prior SSRF fix:** [#29](https://github.com/strands-agents/mcp-server/pull/29) -- initial URL validation
