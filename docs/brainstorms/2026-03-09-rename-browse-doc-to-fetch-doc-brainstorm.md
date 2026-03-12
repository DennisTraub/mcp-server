# Brainstorm: Rename browse_doc to fetch_doc

**Date:** 2026-03-09
**Context:** PR #32 on strands-agents/mcp-server — reviewer feedback from @mkmeral
**PR:** https://github.com/strands-agents/mcp-server/pull/32

## What We're Building

A pure mechanical rename of the `browse_doc` MCP tool back to `fetch_doc`, plus formatting fixes via `hatch fmt`. This addresses the final review feedback on PR #32 before merge.

**Why:** Existing users have `fetch_doc` in their MCP `allowedTools` / `autoApprove` configurations. Introducing a new tool name (`browse_doc`) would silently break those configurations — users would need to update their config files to use the new tool. Keeping the `fetch_doc` name preserves backward compatibility.

## Why This Approach

The rename is the simplest path to unblock the PR merge. The tool's behavior (TOC mode, section mode, preamble, smart sectioning) stays identical — only the exposed name changes.

## Key Decisions

1. **Pure rename, no behavioral changes** — No docstring rewording, no logic changes. The function name, all references in tests, comments, README configs, and encoded install URLs all change from `browse_doc` to `fetch_doc`.

2. **Rename the integration test file** — `tests_integ/test_browse_doc.py` becomes `tests_integ/test_fetch_doc.py` for full consistency.

3. **Update all autoApprove references** — README config examples (Kiro, Q Developer CLI, Cursor) and the encoded Kiro install button URL all get updated. Keeps the branch self-consistent.

4. **Two-commit sequence** — (1) Rename commit, (2) Formatting commit via `hatch fmt`. Clean separation of concerns for easy review.

5. **Use hatch for formatting and linting** — Install hatch and run `hatch run format` + `hatch run lint` (defined in pyproject.toml as `hatch fmt --formatter` and `hatch fmt --linter`). CI runs `hatch run lint` on PRs. No `prepare` script exists in this repo — @mkmeral's `hatch run prepare` likely means "run format + lint".

## Blast Radius

| Category | Files | What changes |
|----------|-------|-------------|
| Server code | `server.py` | Function name `browse_doc` -> `fetch_doc`, 2 self-referencing `browse_doc(uri=...)` calls in the "Recommended workflow" docstring (lines 105-106), tool description text |
| Unit tests | `test_server.py`, `conftest.py` | Import name, all call sites, comments |
| Integration tests | `test_browse_doc.py` -> `test_fetch_doc.py` | File rename + import/call sites |
| README | `README.md` | 5 occurrences in config examples + 1 encoded URL in Kiro badge |
| Text processor | `text_processor.py` | No changes (no `browse_doc` references) |

## Open Questions

None — all decisions resolved during brainstorming.
