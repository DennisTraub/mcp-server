---
title: Rename browse_doc to fetch_doc
type: refactor
status: completed
date: 2026-03-09
origin: docs/brainstorms/2026-03-09-rename-browse-doc-to-fetch-doc-brainstorm.md
---

# refactor: Rename browse_doc to fetch_doc

## Overview

Pure mechanical rename of the `browse_doc` MCP tool back to `fetch_doc` to preserve backward compatibility for existing users who have `fetch_doc` in their MCP `allowedTools` / `autoApprove` configurations. Addresses final reviewer feedback on PR #32 from @mkmeral. No behavioral changes.

## Problem Statement

The `feat/browse-doc` branch renamed the tool from `fetch_doc` to `browse_doc`. This would silently break existing user configurations that reference `fetch_doc` (see brainstorm: docs/brainstorms/2026-03-09-rename-browse-doc-to-fetch-doc-brainstorm.md).

## Proposed Solution

Global find-and-replace of `browse_doc` -> `fetch_doc` across all source, test, and documentation files, plus a `git mv` for the integration test file. Follow with `hatch fmt` for formatting compliance.

## Blast Radius (Verified)

| File | Occurrences | What changes |
|------|-------------|--------------|
| `src/strands_mcp_server/server.py` | 3 | Function def + 2 docstring self-references. `@mcp.tool()` has no `name=` param, so the MCP tool name derives from the function name. |
| `tests/test_server.py` | 15 | Import, 11 call sites, 3 docstrings |
| `tests/conftest.py` | 3 | Comment references |
| `tests_integ/test_browse_doc.py` -> `test_fetch_doc.py` | 6 + file rename | `git mv` for history preservation, then import + call sites + docstring |
| `README.md` | 5 | 3 autoApprove JSON configs (Kiro, Q CLI, Cursor), 1 tool description, 1 percent-encoded Kiro install URL |

**Total: 32 text occurrences + 1 file rename across 5 files.**

## Acceptance Criteria

- [x] All 32 occurrences of `browse_doc` replaced with `fetch_doc` across the 5 files listed above (includes the percent-encoded Kiro badge URL -- plain text replacement is safe)
- [x] `tests_integ/test_browse_doc.py` renamed to `tests_integ/test_fetch_doc.py` via `git mv`
- [x] `hatch run test` passes (unit tests)
- [x] `hatch run lint` passes (no regressions)
- [x] `hatch fmt` applied for formatting compliance
- [x] No behavioral changes to tool logic, text processing, or URL validation

## Implementation Steps

### Commit 1: refactor: rename browse_doc to fetch_doc

1. `git mv tests_integ/test_browse_doc.py tests_integ/test_fetch_doc.py`
2. Global find-and-replace `browse_doc` -> `fetch_doc` in:
   - `src/strands_mcp_server/server.py`
   - `tests/test_server.py`
   - `tests/conftest.py`
   - `tests_integ/test_fetch_doc.py` (already renamed)
   - `README.md`
3. Grep for `browse_doc` to verify zero remaining hits (covers Kiro URL implicitly)
4. Run `hatch run test` to validate
5. Commit

### Commit 2: style: apply hatch fmt formatting

1. Run `hatch run format` (or `hatch fmt --formatter`)
2. Run `hatch run lint` to validate
3. Commit only if formatting changes were produced

## Risks

| Risk | Mitigation |
|------|------------|
| `hatch fmt` touches unrelated files | Separate commit isolates formatting from the rename |
| Missed occurrence | Grep for `browse_doc` after replacement to verify zero remaining hits |

## Sources

- **Origin brainstorm:** [docs/brainstorms/2026-03-09-rename-browse-doc-to-fetch-doc-brainstorm.md](../brainstorms/2026-03-09-rename-browse-doc-to-fetch-doc-brainstorm.md) -- all key decisions (pure rename, two-commit sequence, hatch formatting) carried forward
- **PR:** https://github.com/strands-agents/mcp-server/pull/32
- **Reviewer:** @mkmeral
