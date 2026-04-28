# BerriAI/litellm #26688 — fix(vertex): drop mixed search tools with functions

- PR: https://github.com/BerriAI/litellm/pull/26688
- Author: onthebed
- Head SHA: `f778b43e6067`
- Diff: 6300+/1500- across **66 files** (the actual fix is ~32 lines in `vertex_and_google_ai_studio_gemini.py`)

## Verdict: request-changes

## Rationale

- **The actual fix is correct, small, and well-tested.** At `litellm/llms/vertex_ai/gemini/vertex_and_google_ai_studio_gemini.py:1226-1255`, a final cleanup pass at the end of `map_openai_params` filters out `googleSearch` / `googleSearchRetrieval` / `enterpriseWebSearch` / `urlContext` tools whenever any tool has a `function_declarations` key, gated on `not include_server_side_tool_invocations`. The 35-line test at `tests/test_litellm/llms/vertex_ai/gemini/test_vertex_web_search_tool_order.py` covers the failing-before-this-fix case (`web_search_options` mapped *before* function tools are added) and asserts the search tool is dropped while function declarations survive. The `verbose_logger.warning` at `:1248-1254` names the four offending tool kinds and tells the caller why — good operator UX.
- **But the PR is unmergeable in current shape** — the diff covers **66 files / +6300 / −1500 lines** including unrelated work that the maintainer cannot have intended to land in a "fix(vertex): drop mixed search tools with functions" PR:
  - `.npmrc` `min-release-age=3d` → `min-release-age=3` reverting the W17 supply-chain mitigation reviewed in drip-136 #26677
  - Entire new XecGuard guardrail (`litellm/proxy/guardrails/guardrail_hooks/xecguard/xecguard.py:585+`, 1904-line test file, dashboard logo SVG, garden configs, type defs, 314-line docs page)
  - `expired_ui_session_key_cleanup_manager.py:156+` brand-new module + 348-line test file
  - Predibase chat handler/transformation refactor (228 deletions in handler, 212 additions in transformation)
  - `circleci/config.yml:226-435` awk-pipeline change to test discovery in three jobs
  - 484-line deletion of `SpendLogsSettingsModal.test.tsx` + 156-line deletion of the component itself
  - `model_prices_and_context_window.json` and `_backup.json` 32+/32- pricing-row diffs
  - Plus dozens more
- **This is a bad rebase or a branch-from-main-instead-of-from-feature shape, not a malicious PR.** The author body (`onthebed`) clearly describes only the vertex fix and references `Fixes #26682`. None of the unrelated changes are mentioned. The maintainer reviewer cannot merge this without inheriting all 66 files of unrelated, unreviewed work — which includes the supply-chain regression of `min-release-age=3d → 3` that previous drip work flagged as an intentional defense.
- **Recommended action:** request-changes asking the author to rebase onto current `litellm_internal_staging` (or whatever the target branch is) and force-push a clean diff containing only the four files actually relevant to the fix (`vertex_and_google_ai_studio_gemini.py`, the new test file, plus optionally the `.github/pull_request_template.md` Linear-ticket addition if that's intentional). The structural fix is correct and worth landing — just not in this shape.

## Nits / follow-ups (apply once rebased)

- `optional_params["tools"] = filtered_tools` at `:1255` mutates `optional_params` only when `len(filtered_tools) != len(tools)` — correct (avoids no-op writes), but the `verbose_logger.warning` should probably also include the request id or model name so operators can correlate the warning to a specific call.
- The `include_server_side_tool_invocations` gate at `:1230` is the right escape hatch (preserves server-side tool invocations like `googleSearchRetrieval` when no function tools are present, even mixed) but no test pins that this flag actually preserves the search tools when set. A second test cell `web_search_options + function_tools + include_server_side_tool_invocations=True → search tools survive` would lock the bypass behavior.
- The four-tool list (`GOOGLE_SEARCH`, `GOOGLE_SEARCH_RETRIEVAL`, `ENTERPRISE_WEB_SEARCH`, `URL_CONTEXT`) duplicates whatever the *first* filter pass uses earlier in `map_openai_params`. Should be a single `VERTEX_SEARCH_TOOL_NAMES` set/iterable on `VertexToolName` so the next added search-tool-kind doesn't get missed by one of the two filter sites.
- The fix gates on `any("function_declarations" in tool for tool in tools)` but Gemini's actual incompatibility is "function-calling tools cannot mix with search tools" — if a future tool kind has function semantics under a different key (e.g. a hypothetical `code_execution` tool), this filter misses it. Worth a comment that "function_declarations" is the *current* discriminator, with an issue link to the upstream Gemini constraint doc.

## What I learned

The fix logic is textbook — a final defensive sweep at the end of param-mapping that catches the order-dependent bug regardless of which path inserted the conflicting tool first. The structural lesson is that `map_openai_params` is a long function with many independent insertion points (web_search_options, function tools, code execution, etc.) and Gemini's actual constraint isn't enforced at any single insertion site, so a final cleanup pass is the only place the *combined* invariant can be checked. This pattern — "validate-and-coerce after all independent transforms have run" — generalizes anywhere you have a target-side constraint that spans multiple source-side fields. The bigger lesson here, though, is the PR-shape one: a 4-file fix submitted as a 66-file diff is a rebase mistake, not a feature, and should never be merged regardless of how good the actual fix is. Catching this in review depends on counting files before reading the diff.
