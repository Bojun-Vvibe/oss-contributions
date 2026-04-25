# charmbracelet/crush #2671 — Reduce `fetch` and `view` truncation size to 100KB

- **Repo**: charmbracelet/crush
- **PR**: [#2671](https://github.com/charmbracelet/crush/pull/2671)
- **Head SHA**: `cea17e69610adf194d348238f77ca4f6ccfa1e59`
- **Author**: andreynering
- **State**: MERGED
- **Verdict**: `merge-as-is`

## Context

Drops the per-call ceilings on the `fetch` and `view` tools from 1MB to
100KB, in the interest of keeping a single tool result from blowing out
the model's context window. Also rewrites the tool descriptions in
`fetch.md` / `view.md` to advertise the new limit. The bulk of the diff
(~7600 lines) is regenerated VCR-style YAML cassettes under
`internal/agent/testdata/TestCoderAgent/glm-5.1/*.yaml` reflecting the
new max-size string in tool descriptions.

## Design

The actual code change is six lines:

- `internal/agent/tools/fetch.go:21` — `MaxFetchSize` constant goes from
  `1 * 1024 * 1024` (1MB) to `100 * 1024` (100KB).
- `internal/agent/tools/view.go:55` — `MaxViewSize` constant goes from
  `1 * 1024 * 1024` to `100 * 1024`.
- `internal/agent/tools/fetch.md:1,32` and `view.md:1,21` — descriptions
  updated from "max 5MB" / "max 1MB" to "max 100KB" in both the
  one-liner and the `<limitations>` block.

The constant-name approach is exactly right: callers reference the named
constant (no magic number duplicated elsewhere), so the truncation behavior
in `fetch.go`/`view.go` flips atomically with the constant. The
prose-level discrepancy where the description previously said "max 5MB"
while the constant was already 1MB is also corrected as a side effect —
nice cleanup.

The cassette regeneration is mechanical: every `bash_tool.yaml`,
`fetch_tool.yaml`, etc. embedded the entire system+tool prompt as a
recorded request body, and the tool descriptions changed, so every
cassette had to re-record. This is why the diff looks huge but the
behavior change is six lines.

## Risks / Nits

1. **100KB is a hard cap, not a soft cap.** The change directly affects
   the `fetch` tool — anyone fetching a >100KB doc page now gets it
   truncated at 100KB with no spillover handling. For the `view` tool the
   default `DefaultReadLimit = 2000` lines and `MaxLineLength = 2000`
   chars at `view.go:56-57` already provide a separate truncation axis,
   but for `fetch` this is the only ceiling. Worth confirming with a
   release note — users who'd built workflows around fetching mid-size
   reference docs (e.g., 300KB markdown specs) will get silently
   truncated outputs.

2. **No companion change to add an `offset`/pagination knob to `fetch`.**
   `view` already has `offset` (`view.go` `DefaultReadLimit`/`offset`
   pattern) so a user can re-`view` past the truncation point. `fetch`
   has no such escape hatch in the current diff — once truncated, the
   tail is unreachable without a different tool. The reduction itself is
   defensible; the absence of a pagination story for `fetch` is the
   thing that will surface as a follow-up issue.

3. **No code-side test asserts the new constant value.** A trivial
   `assert.Equal(t, 100*1024, MaxFetchSize)` in
   `internal/agent/tools/fetch_test.go` (or the existing test file)
   would prevent silent regression if someone bumps the constant later
   without updating the prose.

## What I learned

The cassette explosion (>7600 lines) on a 6-line behavior change is
worth pausing on as a project-health observation: when tool descriptions
are part of the recorded test fixtures, every prose tweak forces a full
re-record. That's not wrong — it's the cost of locking down model
prompts as test data — but it does make the actual code change hard to
spot in review. A `git diff -- ':!internal/agent/testdata'` view of this
PR fits in 30 lines and reads as obviously correct.
