# sst/opencode PR #25640 — fix: allow Codex Spark with Codex OAuth

- Link: https://github.com/sst/opencode/pull/25640
- SHA: `5c3a4b5da3d3e2f4c343838ff8b4a711b21ab1c6`
- Author: Utkub24
- Stats: +8 / −1, 1 file

## Summary

Single-line allowlist addition: `gpt-5.3-codex-spark` is added to the `ALLOWED_MODELS` set in `packages/opencode/src/plugin/codex.ts` so the Codex OAuth flow accepts it alongside the existing `gpt-5.5`, `gpt-5.2`, `gpt-5.3-codex`, `gpt-5.4`, `gpt-5.4-mini`. Closes #25638.

## Specific references

- `packages/opencode/src/plugin/codex.ts` L17–L24: the diff replaces the single-line `new Set([...])` literal with a multi-line literal and inserts `"gpt-5.3-codex-spark"` between `"gpt-5.3-codex"` and `"gpt-5.4"`. The reformatting also makes future allowlist additions a clean single-line diff, which is a small but real maintainability win.
- L17 (set construction): `ALLOWED_MODELS` is a `Set<string>` constructed once at module load. No memoisation concerns; `.has()` lookup remains O(1).
- The constant is module-private (no `export`) and there is no schema/config file or docs that enumerates the allowed model ids elsewhere in the diff — consumer-visible surface is solely the runtime check. If the broader project keeps a docs page listing OAuth-supported models, that page should also mention `gpt-5.3-codex-spark` (out of scope for this PR but worth noting in the PR description).
- No tests were added or modified. For a pure allowlist entry this is consistent with how prior model additions in this file have been handled.

## Verdict

verdict: merge-as-is

## Reasoning

Trivial allowlist entry that fixes a reported regression (#25638). The change is one new string in a `Set`, the surrounding code is reformatted into a multi-line literal which improves future diffs, and there is no behavioural risk beyond admitting one additional model id into an OAuth path that already accepts five other Codex-family models. Author confirms local verification.
