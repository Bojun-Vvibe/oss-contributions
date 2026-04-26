---
pr: 2675
repo: charmbracelet/crush
sha: a0bd0773aa018de1128a08dbeb57bbcf90368940
verdict: merge-after-nits
date: 2026-04-27
---

# charmbracelet/crush #2675 — Fix command aliases/args parsing and empty tool-call input normalization

- **Head SHA**: `a0bd0773aa018de1128a08dbeb57bbcf90368940`
- **Size**: bundled — touches `internal/agent/agent.go`, `internal/commands/commands.go`, plus tests, plus `internal/message/content.go`

## Summary
Three independent fixes packaged together:
1. **Tool-call input normalization** (`internal/agent/agent.go:387, 618-624`) — wraps stored `tc.Input` in a new `normalizeToolCallInput()` that returns `"{}"` when the provider sends an empty/whitespace-only input string. Prevents `json.Unmarshal("")` failures downstream.
2. **Command named-arg regex Unicode support** (`internal/commands/commands.go:17`) — replaces the ASCII-only `\$([A-Z][A-Z0-9_]*)` with `\$([\p{L}_][\p{L}\p{N}_]*)`, allowing `$数据目录`, `$ДАННЫЕ_1`, `$ÅR2`. Tested in `commands_test.go:55-67`.
3. (Implied — not fully visible) edits in `internal/message/content.go` likely related to the tool-call normalization.

## Specific findings
- `agent/agent.go:618-624` — `normalizeToolCallInput` is a clean four-line function: `strings.TrimSpace(input) == ""` → return `"{}"`, else passthrough. The test `agent_test.go:797-803` covers `""`, all-whitespace, and a real JSON object. Good. Missing case: a literal `"null"` input string. Some providers send `"null"` for no-arg tools; that would currently flow through unchanged and fail `json.Unmarshal` at the consumer if it expects an object. Worth one extra branch.
- `commands/commands.go:17` — the new regex `\$([\p{L}_][\p{L}\p{N}_]*)` matches Unicode-letter-then-letter-or-digit identifiers. **Behavior change**: previously `$abc` would not match (lowercase rejected), now it does. If existing user command files rely on `$lowercase` being treated as literal text (because the old regex required uppercase), this is a silent breaking change. Users can escape with `\$lowercase` if needed but PR body doesn't document the change.
- `commands_test.go:54-66` — `TestExtractArgNames_UnicodePlaceholders` is the one test added; covers `$DATA_DIR`, `$数据目录`, `$ДАННЫЕ_1`, `$ÅR2`. Notably absent: a test that confirms `$lowercase` *now* matches (the breaking-change case). Also no test for `$1` (digit-first, must still be rejected by the leading `[\p{L}_]` class — good but should be asserted).
- The PR title combines two unrelated concerns ("command aliases/args parsing" + "empty tool-call input normalization"). Splitting would make bisect easier and let the regex change get its own discussion thread.

## Risk
Low for normalization (fail-safe default). Medium for the regex change (silent semantic widening). The lowercase-now-matches case is the one most likely to hit existing user command files.

## Verdict
**merge-after-nits** — split the PR or at least call out the lowercase-matches behavior change in the body; add tests for `$lowercase` and `$1`; consider also normalizing `"null"` in `normalizeToolCallInput`.
