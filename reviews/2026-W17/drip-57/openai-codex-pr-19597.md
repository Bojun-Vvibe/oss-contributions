# openai/codex #19597 — Fix TUI attach fallback test stack overflow

- URL: https://github.com/openai/codex/pull/19597
- Head SHA: `68f714b7c0f8c69d3663dc92fe9379b6bb43f972`
- Verdict: **merge-as-is**

## What it does

Two `attach_live_thread_for_selection_rejects_*` tests in
`codex-rs/tui/src/app/tests.rs` exercise the embedded app-server path,
which can blow the default Rust test-thread stack in debug builds —
reproduced on Windows in #19596 and locally without setting
`RUST_MIN_STACK`. The fix runs each affected test on a manually-spawned
thread with an explicit 4 MiB stack.

## Diff notes

The new helper `run_stack_heavy_app_server_test` at
`codex-rs/tui/src/app/tests.rs:1352-1374` builds a 4 MiB stack thread,
spins up a current-thread Tokio runtime inside it, and joins the
result:

```
let handle = std::thread::Builder::new()
    .name(test_name.to_string())
    .stack_size(TEST_STACK_SIZE_BYTES)   // 4 MiB
    .spawn(move || -> Result<()> {
        let runtime = tokio::runtime::Builder::new_current_thread()
            .enable_all()
            .build()?;
        runtime.block_on(test_body())
    })?;
```

Both affected tests change from `#[tokio::test] async fn ...` to
`#[test] fn ...` and pipe their bodies through the helper. The test
bodies themselves are unchanged — same `attach_live_thread_for_selection`
calls, same `expect_err`, same `assert!` on
`thread_event_channels`. Good — fixes the runner without rewriting the
behavior under test.

## Risk surface

- `block_on` on a current-thread runtime is the right call here since
  the test bodies don't fan out work; no risk of starving multi-thread
  scheduling.
- 4 MiB matches what the team has elsewhere for stack-heavy paths and
  leaves headroom for debug-build frame inflation.
- Panics inside the test body propagate via `handle.join()` and get
  rewritten as `eyre!("{test_name} thread panicked")`. Slightly less
  rich than the default `#[tokio::test]` panic surface, but acceptable
  for two specific tests.

## Why this verdict

Surgical fix to two specific test cases that were blocking CI in debug
builds. No production code touched. Lands clean.
