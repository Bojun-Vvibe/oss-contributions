# sst/opencode #25917 — fix(shell): advertise actual default timeout in tool description

- URL: https://github.com/sst/opencode/pull/25917
- Head SHA: `78eacba8fce1509cb42c860a0a7e54ef46201f29`
- Author: nabilfreeman
- Size: +5 / -4 across 2 files

## What it does

Replaces the hardcoded `"will time out after 120000ms (2 minutes)"` string in the shell-tool prompt template with `${limits.defaultTimeoutMs}ms` interpolation, so the LLM-facing tool description always matches the actual `DEFAULT_TIMEOUT` constant (currently 120000ms but historically tweaked). Adds a new `defaultTimeoutMs: number` field to the existing `Limits` type at `prompt.ts:20` and threads it through from the call site at `shell.ts:588` (`{ ...limits, defaultTimeoutMs: DEFAULT_TIMEOUT }`). Three template strings are updated identically across the bash, powershell, and unspecified-shell branches (`prompt.ts:106`, `:152`, `:202`).

## Observations

- The fix is structurally correct: `DEFAULT_TIMEOUT` lives in `shell.ts`, `Limits` is constructed in `shell.ts`, the prompt is rendered from `shell.ts` — so spreading `defaultTimeoutMs: DEFAULT_TIMEOUT` into the `limits` object at the render site keeps the source-of-truth single. The previous string drift (template said "120000ms" while `DEFAULT_TIMEOUT` could in principle be anything) is now structurally impossible.
- One mild nit: the pretty-print "(2 minutes)" suffix is dropped in favor of raw milliseconds. For 120000ms the human-readable annotation was useful context for the model. A `${limits.defaultTimeoutMs}ms (${Math.round(limits.defaultTimeoutMs/60000)} minutes)` would preserve the original UX, but only matters if someone reads the tool description directly.
- All three updated lines are byte-identical changes; no risk of one shell branch diverging from another.
- The `Limits` type is exported, so any external consumer constructing a `Limits` literal must now supply `defaultTimeoutMs` — minor breaking change for the package's public surface, but `Limits` reads as an internal type and there are no other call sites in the diff.

## Verdict: merge-as-is

Tight 9-line documentation-correctness fix that eliminates a maintenance footgun (the prompt and constant could silently drift). The minor "(2 minutes)" suffix loss is cosmetic and not worth blocking on.