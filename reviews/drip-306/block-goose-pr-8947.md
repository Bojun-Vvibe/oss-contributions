# Review: block/goose #8947 — feat(acp): add GOOSE_DISABLE_TOOL_CALL_SUMMARY

- **Repo**: block/goose
- **PR**: #8947
- **Head SHA**: `97b84239c541441a6247bd96783ace4bca142dbe`
- **Author**: ocervinka

## What it does

Adds an env var (`GOOSE_DISABLE_TOOL_CALL_SUMMARY`) to opt out of the
per-tool-call AI-generated title shown in the ACP UI. When disabled,
the synchronous fallback title (tool name + truncated args) is kept
and the auxiliary `provider.complete_fast` call per tool invocation is
skipped. Motivated by rate-limited providers (Code Assist OAuth,
Gemini API free tier) where the extra round-trip exhausts quota.

## Diff notes

- `crates/goose/src/acp/server.rs:1432-1444` — early-returns out of the
  summary-generation block right after the initial `SessionUpdate::ToolCall`
  is sent (so the UI still gets the fallback title). Placement is correct:
  before the `provider.complete_fast` spawn, after the synchronous fallback
  was already emitted.
- `crates/goose/src/config/base.rs:1022` — adds
  `config_value!(GOOSE_DISABLE_TOOL_CALL_SUMMARY, bool);` next to the
  sibling `GOOSE_DISABLE_SESSION_NAMING` flag. Naming and shape match
  the existing pattern exactly.
- `documentation/docs/guides/environment-variables.md:235,277-279` —
  adds a thorough table row and a usage example. The motivation
  paragraph is clear and accurate (cites Code Assist OAuth and Gemini
  free tier as the rate-limit offenders).

## Concerns

1. **No test**. The `Config::global().get_goose_disable_tool_call_summary()`
   gate is trivial but uncovered. A unit test that mocks `Config` and
   asserts the spawn is skipped when the flag is true would make this
   regression-proof. Non-blocking but recommended.
2. **Default truthy value**: `unwrap_or(false)` — correct, preserves
   today's behavior. Good.
3. **Spelling**: "Tool Call Summary" vs the upstream code path which
   talks about *titles* (`fallback_title`, `complete_fast` for title
   upgrade). The env var name says "summary" but the docs say "title".
   Minor inconsistency that could confuse users when they grep. Either
   rename to `GOOSE_DISABLE_TOOL_CALL_TITLE` (cleaner) or update the docs
   to consistently use "summary".
4. **Naming parity**: `GOOSE_DISABLE_SESSION_NAMING` exists for a
   sibling problem; this one mirrors it well. Consider mentioning the
   sibling in the docs ("If you're already disabling
   `GOOSE_DISABLE_SESSION_NAMING` for the same reason, you'll likely want
   this too").

Mechanically clean, scoped, gated, and documented. Pure additive
behavior.

## Verdict

merge-after-nits
