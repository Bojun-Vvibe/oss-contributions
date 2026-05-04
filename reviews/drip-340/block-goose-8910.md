# Review: block/goose #8910 — fix(cli): report cumulative total_tokens in stream-json/json output

- **PR**: https://github.com/block/goose/pull/8910
- **Author**: bzqzheng
- **Base**: `main`
- **Head SHA**: `0b54c1d2a312de1cec0478df35650310d22ca94b`
- **Size**: +199 / −3 across 2 files
- **Fixes**: #8871

## Scope

The CLI's `--json` and `--stream-json` output modes were reading `session.total_tokens`, which the agent code only ever stores as the **last turn's** usage. This caused `complete.total_tokens` in the streaming JSON stream to reset to the current turn's count on every chunk, instead of the running session total. Fix swaps three call sites to `session.accumulated_total_tokens` (which `update_session_metrics` in `crates/goose/src/agents/reply_parts.rs` already maintains correctly) and adds a multi-turn integration test proving the cumulative behaviour.

## Substantive review

### `crates/goose-cli/src/session/mod.rs` (+3 / −3)

Three small, surgical replacements:

- **Line 1268** (`JsonMetadata` construction inside `--json` output): `total_tokens: session.total_tokens` → `total_tokens: session.accumulated_total_tokens`. Correct — JSON output should always reflect the session-wide total.
- **Line 1289** (stream-JSON `Complete` event emit): `.and_then(|s| s.total_tokens)` → `.and_then(|s| s.accumulated_total_tokens)`. Correct — this is the field that was misbehaving in the bug report.
- **Line 1448** (`get_total_token_usage` public-ish helper): `metadata.total_tokens` → `metadata.accumulated_total_tokens`. Correct — the function name explicitly says "total token usage", which can only mean cumulative.

There is no behaviour-preserving wrapper or migration shim because the existing CLI never stored its result anywhere other than these three local consumers (all changed in this PR). The `session.total_tokens` field still exists and continues to mean "last turn"; nothing else in this file references it after this PR.

### `crates/goose/tests/agent.rs` (+196)

Adds a `cumulative_token_tests` module with:

- `FixedUsageProvider`: a mock `Provider` impl that reports a fixed `(input=10, output=5, total=15)` `ProviderUsage` on every `stream()` call, with an `AtomicUsize` call counter for sanity.
- `test_accumulated_total_tokens_across_multiple_turns`:
  - Creates a fresh `SessionManager` in a `tempfile::tempdir()`.
  - Runs three sequential `agent.reply(...)` turns with `max_turns: Some(1)` each.
  - After each turn, asserts `session.accumulated_total_tokens == Some(15 * n)` (15, 30, 45).
  - **Critically also asserts** `session_after_3.total_tokens == Some(15)` — i.e. the last-turn-only field still reads as 15. This locks in the invariant that `total_tokens` is per-turn and `accumulated_total_tokens` is cumulative, preventing future regressions in either direction.

The test design is exactly right. A few small observations:

- `FixedUsageProvider::call_count` is incremented on every `stream()` call but never asserted in the test — would be a one-line addition (`assert_eq!(provider.call_count.load(Ordering::SeqCst), 3)`) and would catch the case where some retry path called the provider an unexpected number of times.
- The mock returns a fresh `ProviderUsage::new("mock-model", ...)` on every call. Worth documenting in a comment that this models the realistic case where the provider reports per-turn usage and goose accumulates server-side.
- The test files a `SessionType::Hidden` session with a default `PathBuf` working directory; please confirm `Hidden` doesn't take a special accumulation path that bypasses normal usage tracking. (If a `SessionType::User` test is cheap to add as well, that would belt-and-braces this.)

## Risk

- This is a CLI-output bugfix with no schema change to anything persisted or to the public Rust API. The risk surface is the three swapped field reads.
- Anyone scripting against the previous (buggy) `total_tokens` value in JSON output and relying on it being the per-turn value will see a behaviour change. That's the intended fix per #8871, but the CLI changelog/release notes should call it out so dashboards aren't silently broken.

## Verdict

**merge-as-is** — minimal, correct, and the test is exactly the right shape. Optional nits (call-count assertion, mock-comment, additional `SessionType::User` case) can land as a follow-up. The release notes should explicitly mention that `complete.total_tokens` in `--stream-json` now reports the cumulative session total.
