# openai/codex PR #19453 — Serialize legacy Windows PowerShell sandbox tests

- **Repo:** openai/codex
- **PR:** [#19453](https://github.com/openai/codex/pull/19453)
- **Head SHA:** `b55d10abc98a19de069cd07c19fc8bb3d67b7595`
- **Author:** dylan-hurd-oai
- **Size:** +13 / −0 across 1 file
- **Reviewer:** Bojun (drip-27)

## Summary

Addresses a cluster of 30s timeouts in the legacy Windows sandbox suite
(`legacy_capture_powershell_emits_output`,
`legacy_tty_powershell_emits_output_and_accepts_input`,
`legacy_non_tty_cmd_emits_output`,
`legacy_non_tty_powershell_emits_output`) by serializing the four tests
behind a single `Mutex<()>` guard. PR body links six failing CI runs that
all hit the same timeout pattern. Fix is one file:
`codex-rs/windows-sandbox-rs/src/unified_exec/tests.rs`.

## Key changes

### `tests.rs:30–37` — shared lock

```rust
static LEGACY_PROCESS_TEST_LOCK: Mutex<()> = Mutex::new(());

fn legacy_process_test_guard() -> MutexGuard<'static, ()> {
    LEGACY_PROCESS_TEST_LOCK
        .lock()
        .expect("legacy Windows sandbox process test lock poisoned")
}
```

A process-wide `std::sync::Mutex` plus a tiny accessor that panics on
poisoning. `MutexGuard` is bound to `'static` because the lock itself is
static, which keeps the guard valid for the entire test body.

### `tests.rs:141, 180, 365, 402` — guard placements

Each of the four failing tests now starts with
`let _guard = legacy_process_test_guard();` *before* spinning up the
current-thread runtime and `block_on(...)`. Because the guard is held by
the synchronous outer scope and never moved into the async task, it
covers the entire `block_on` and is dropped when the test function
returns. The two `pwsh_path()`-skipped tests acquire the guard *after*
the early-return, so a host without PowerShell still skips cleanly
without ever taking the lock.

## What's good

- The blast radius is exactly four functions and zero production code.
  No timeout bumps, no `sleep`s, no test deletions — just sequencing the
  tests that share the legacy Windows sandbox global state.
- Acquiring the guard *after* the `pwsh_path()` skip avoids a needless
  lock acquisition (and a needless serialization point) on hosts that
  don't have PowerShell installed.
- Putting the guard before `current_thread_runtime()` rather than inside
  the `async move` block means the guard's lifetime is bounded by the
  sync outer scope, sidestepping the "MutexGuard isn't `Send`" footgun
  that would surface immediately if anyone later switched these to a
  multi-thread runtime.
- The PR body cites the exact failing runs (24909500958, 24908076251,
  24906197645, 24905411571, 24903336028, 24898949647), which makes the
  diagnosis auditable rather than speculative.

## Concerns

1. The fix treats the symptom (concurrent test runs racing on shared
   legacy-sandbox state) but doesn't document *what* the shared state
   is. A two-line comment above `LEGACY_PROCESS_TEST_LOCK` explaining
   why these four tests in particular need serialization (e.g. "they
   share a per-process pwsh job object" / "they each spawn the legacy
   capture pipe and the OS only allows one outstanding handle") would
   stop a future contributor from removing the lock or from forgetting
   to add it to a fifth legacy test that gets added later.

2. `expect("...lock poisoned")` is fine in test code, but if any of the
   four tests panics while holding the lock, every subsequent legacy
   test in the same process will also panic with this message instead
   of getting a chance to run and report its own failure. For tests
   this is usually preferable (poison = something is fundamentally
   broken), but it does mean the first failure in the cluster will
   mask the others. Worth a one-line comment so the next person
   debugging a poisoned-mutex panic knows to look at the *first*
   failure in the run, not this one.

3. No new test added to verify the serialization itself works. That's
   reasonable — verifying "this lock prevents a race that only happens
   on Windows CI" is hard to do in a portable way — but it does mean
   the next regression will only surface on the same Windows runners.

## Risk

Very low. Test-only change, behind `#[cfg]` for Windows targets. Worst
case the lock is unnecessary and the four tests run sequentially when
they could have run in parallel (a few extra seconds of CI time).

## Verdict

**merge-as-is**

The diagnosis matches the linked CI evidence, the fix is minimal and
correctly scoped, and the lifetime/early-return ergonomics are right.
The two documentation nits above are improvements, not blockers.

## What I learned

`std::sync::Mutex<()>` as a process-wide test serializer is the cleanest
pattern for "these tests share OS-level singleton state" — much better
than the common alternatives of `#[serial]` macros (extra dep) or
splitting the tests into separate binaries (extra build time). The key
detail is binding the `MutexGuard` to a `let _guard` in the synchronous
outer scope so it lives across the `block_on` boundary; if you put it
inside the async block it has to be `Send`, which `MutexGuard` isn't on
all platforms.
