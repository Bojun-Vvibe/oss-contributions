# block/goose PR #8947 — feat(acp): add GOOSE_DISABLE_TOOL_CALL_SUMMARY to opt out of per-tool-call summaries

- URL: https://github.com/block/goose/pull/8947
- Head SHA: `97b84239c541441a6247bd96783ace4bca142dbe`
- Verdict: **merge-as-is**

## Summary

Adds an opt-out environment variable, `GOOSE_DISABLE_TOOL_CALL_SUMMARY`,
that short-circuits the per-tool-call `complete_fast` provider request
that would otherwise upgrade the synchronous fallback title (tool name
+ truncated args) into a model-generated phrase. When set, the fallback
title is kept and the auxiliary call is skipped — useful on
rate-limited providers (Code Assist OAuth, Gemini API free tier) where
the extra round-trip per tool call exhausts quota.

## Specific references

- `crates/goose/src/acp/server.rs:1431-1445` — the new early-return:

  ```rust
  if Config::global()
      .get_goose_disable_tool_call_summary()
      .unwrap_or(false)
  {
      return Ok(());
  }
  ```

  Placed *after* the synchronous `SessionUpdate::ToolCall(initial_tool_call)`
  send, so the UI still receives the fallback title; only the
  asynchronous AI-summary upgrade is skipped. This is the correct
  insertion point.

- `crates/goose/src/config/base.rs:1022` — registers the new key via
  the existing `config_value!(GOOSE_DISABLE_TOOL_CALL_SUMMARY, bool)`
  macro, immediately adjacent to `GOOSE_DISABLE_SESSION_NAMING`. The
  pattern parallel is intentional and good.

- `documentation/docs/guides/environment-variables.md:235` — adds the
  table row with a clear description, valid values (`"1"`, `"true"`
  case-insensitive), default `false`, and the rate-limited-provider
  motivation. Plus a usage example block at line ~277.

## Commentary

This is a textbook "small ergonomic config flag" PR. Three things make
it good:

1. **Correct insertion point.** The early-return happens after the
   initial `SessionUpdate::ToolCall(initial_tool_call)` send. That
   means the UI still gets a title — just the synchronous fallback
   instead of the AI-upgraded one. If the early-return had been
   placed *before* the initial send, the user would see no title at
   all, which would be a regression. The author put it in the right
   place.

2. **Conservative default.** `unwrap_or(false)` preserves existing
   behavior for everyone who doesn't set the variable. No surprise
   for current users.

3. **Documented motivation.** The env-vars table entry explicitly
   names "Code Assist OAuth, Gemini API free tier" as the use case.
   That's the kind of context that prevents a future maintainer from
   ripping the flag out as "dead config."

A few tiny things I'd note but wouldn't block on:

- **No test.** The whole config-flag mechanism is exercised by the
  existing `config_value!` macro tests, and the early-return is one
  line, so a dedicated test is arguably overkill. Still, a unit test
  asserting that `complete_fast` is *not* called when the env var is
  set would be nice insurance.

- **Comment in the source.** The 5-line block comment explaining
  *why* we exit here is genuinely helpful — it explicitly calls out
  the rate-limit failure mode. Good documentation hygiene.

- **Naming consistency.** `GOOSE_DISABLE_TOOL_CALL_SUMMARY` parallels
  `GOOSE_DISABLE_SESSION_NAMING` perfectly. ✓

Merge as-is.
