# openai/codex#20176 — `TUI: Remove core protocol dependency [5/7]`

- URL: https://github.com/openai/codex/pull/20176
- Head SHA: `5564c076b3da`
- Author: etraut-openai
- Size: +2 / -2 in `codex-rs/tui/src/onboarding/auth.rs`
- Stack: 5 of 7 in the "remove direct `codex_protocol::protocol`
  usage from `codex-tui`" series

## Summary

Onboarding auth tests had one lingering `use codex_protocol::protocol::SessionSource;`
import after the rest of the TUI had been migrated to the
app-server `SessionSource` equivalent. This PR removes that last
import in the test module and constructs the `SessionSource` value
via JSON deserialization instead.

## Specific reference

`codex-rs/tui/src/onboarding/auth.rs:1016-1050` (test module, post-patch):

```rust
- use codex_protocol::protocol::SessionSource;
  use pretty_assertions::assert_eq;
  ...
- session_source: SessionSource::Cli,
+ session_source: serde_json::from_value(serde_json::json!("cli"))
+     .expect("cli session source should deserialize"),
```

## Risks

- The new construction is `serde_json::from_value(json!("cli"))` —
  it relies on the field type implementing `Deserialize` for a
  bare string, with the contract "the canonical wire form for the
  CLI variant is the literal string `"cli"`". That's brittle:
  anyone renaming the variant or adding `#[serde(rename_all = ...)]`
  silently breaks this test at runtime instead of compile time.
- It would be cleaner to import the app-server enum directly:
  `use codex_app_server_protocol::SessionSource;` and then
  `session_source: SessionSource::Cli,`. The PR description says
  the rest of onboarding moved to the app-server type, so the
  enum should already be in scope or trivially importable. Using
  JSON-string construction in tests defeats half the type system.

## Verdict

`merge-after-nits`

## Suggested nit

Replace the `serde_json::from_value(json!("cli"))` call with a
direct constructor of the app-server `SessionSource` enum (the
same type used non-test elsewhere after this stack lands). That
keeps the cargo-check verification meaningful and avoids hiding
a future rename behind a runtime panic.

Otherwise the change is mechanical and matches the stated stack
goal of confining `codex_protocol::protocol` usage out of `codex-tui`.
