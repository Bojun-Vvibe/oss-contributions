# PR #8947 — feat(acp): add GOOSE_DISABLE_TOOL_CALL_SUMMARY

- Repo: block/goose
- Head: `97b84239c541441a6247bd96783ace4bca142dbe`
- URL: https://github.com/block/goose/pull/8947
- Verdict: **merge-as-is**

## What lands

Adds a single env-var/config-key opt-out (`GOOSE_DISABLE_TOOL_CALL_SUMMARY`,
default `false`) that skips the per-tool-call `provider.complete_fast`
auxiliary call that upgrades the synchronous `fallback_title` to a model-
generated phrase. The motivation is concrete and well-documented: Code
Assist OAuth has a per-second cap, Gemini API free tier has a daily cap,
and a 4-tool turn fires 4 extra requests for a UI nicety.

## Specific findings

- `crates/goose/src/acp/server.rs:1434-1444` gates the auxiliary call
  cleanly:
  ```
  send_session_update(SessionUpdate::ToolCall(initial_tool_call))?;
  if Config::global()
      .get_goose_disable_tool_call_summary()
      .unwrap_or(false) {
      return Ok(());
  }
  if let Ok(tool_call) = &tool_request.tool_call { ... spawn complete_fast ... }
  ```
  The `ToolCall` notification with the synchronous fallback title is sent
  *before* the gate check, so clients still get the tool-call event with a
  usable title — they just don't get the follow-up `ToolCallUpdate` that
  upgrades the title. Behavior on the client side gracefully degrades.
- `crates/goose/src/config/base.rs:1022` registers the new key via the
  same `config_value!` macro entry as `GOOSE_DISABLE_SESSION_NAMING` two
  lines above. Macro consistency means it picks up env-var lookup, YAML
  config, and the `get_*()` accessor automatically — no missed wire-ups.
- `documentation/docs/guides/environment-variables.md:235,277-279`
  documents both the table entry and an example block. The table entry
  explicitly calls out the rate-limited use cases ("Code Assist OAuth,
  Gemini API free tier") so users hitting the symptom can find the
  workaround.

## Verdict rationale: merge-as-is

- The gate is opt-in and defaults to current behavior, so no risk of
  silent regression for users who never set the env var.
- The PR mirrors `GOOSE_DISABLE_SESSION_NAMING` line-for-line — same
  macro, same access pattern, same docs shape — which is the right way
  to add knobs of this class.
- The author's note about not adding a unit test ("the existing
  `GOOSE_DISABLE_SESSION_NAMING` opt-out ships without one") is a
  defensible appeal to local convention. A test would be cheap to add
  but I wouldn't block merge on it; the gate is one `if` statement on a
  config lookup, and the failure mode (skip on true / proceed on false)
  is hard to get wrong.
- Worth flagging for the maintainers: the underlying issue (auxiliary
  `complete_fast` calls eating provider quota) is real and the env-var
  opt-out is the right tactical fix. A longer-term followup might be to
  batch / dedupe these calls or emit them lazily on UI demand, but
  that's a separate piece of work.
