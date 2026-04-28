# block/goose #8790 — fix: `goose configure` exits silently at the model picker

**Verdict:** `merge-after-nits`

- Repo: `block/goose`
- PR: https://github.com/block/goose/pull/8790
- Head SHA: `7de9ad31`
- Closes: #8373
- Diff: +5 / -0, single file `crates/goose-cli/src/commands/configure.rs`

## What it does

Fixes a real "the entire configure flow exits silently with code 2 right
after `Model fetch complete`" bug on macOS 26.4 / kitty / Terminal.app.
The model picker drew one frame and disappeared before any keypress was
registered. Root cause is a documented race: the `cliclack` spinner's
render thread outlives `spin.stop()` for a few milliseconds, and the next
prompt's `interact()` puts stdin into raw mode and reads. Both widgets
fight over the terminal; the picker's first read returns `Err`, which
propagates as exit 2.

## Design analysis

The fix is the smallest possible change that addresses the race:

```rust
spin.stop(style("Model fetch complete").green());
+    // cliclack's spinner render thread outlives `stop()` and races the next
+    // prompt's raw-mode stdin read; drop and yield so the picker's
+    // `interact()` doesn't fail before any key is read. See #8373.
+    drop(spin);
+    tokio::time::sleep(std::time::Duration::from_millis(50)).await;
```

(`crates/goose-cli/src/commands/configure.rs:743-747`)

Two operations: explicit `drop(spin)` to fire the destructor that joins
the render thread, then a 50ms `tokio::time::sleep` to give the join
enough wall time before the picker grabs raw mode. This is the right
shape for a known upstream-library race that the consumer can't fix
inside the library: synchronize at the consumer boundary, document the
race in a load-bearing comment that points at the issue, and move on.

The comment is good — it names the race ("render thread outlives
`stop()`"), names the affected next operation ("`interact()`"), and
references the bug. A future reviewer wondering "why is there a
`tokio::sleep` in a configure flow" gets the answer in three lines.

The 50ms constant is a reasonable choice: long enough to cover a typical
render-thread teardown on a slow terminal emulator, short enough that the
user perceives it as the spinner naturally finishing rather than as a
stall.

## Risks

1. **The 50ms is a magic number with no upper-bound justification.** If a
   future cliclack version takes >50ms to tear down the render thread (e.g.
   on a heavily loaded CI runner), the race re-opens silently. The fix
   pattern is correct but fragile to upstream timing changes. A more
   robust shape would be a poll loop: "wait until terminal is no longer
   being written to, up to 500ms" — but that's significantly more code
   for a low-recurrence problem.
2. **Acknowledging the underlying upstream bug is not part of the PR.**
   The right long-term fix is `cliclack::Spinner::stop()` becoming
   synchronous (joining the render thread before returning). Worth a
   tracking issue at upstream cliclack with a link back to #8373 so the
   workaround can be removed when fixed upstream. The code comment
   doesn't suggest this; a `// TODO(cliclack): remove once upstream
   stop() is synchronous` would make the technical-debt explicit.
3. **No regression test.** Testing this race deterministically is
   essentially impossible (it depends on terminal timing), so the lack of
   a test is acceptable. But a manual repro recipe in the PR body that
   says "run `goose configure` 10× on macOS 26.4 in kitty, expect picker
   to appear all 10 times" would let future regressions be caught before
   they ship to users again.
4. **`tokio::time::sleep` inside an async fn is correct here**, but if
   `configure_provider_dialog` is ever called from a non-tokio runtime
   (e.g. a sync context wrapper), the 50ms wait silently becomes a
   busy-wait or a panic depending on runtime. The function is already
   `async`, so this is a present-tense correctness statement, not a
   future-tense risk.

## Suggestions

- Promote the magic 50ms to a named constant
  (`SPINNER_TEARDOWN_GRACE: Duration = Duration::from_millis(50)`) at
  module scope so a future reviewer can find / tune it without grepping
  for the literal.
- Add a `// TODO(cliclack): drop this workaround when upstream
  Spinner::stop() blocks on render-thread teardown` line above the
  `tokio::time::sleep` so the technical debt has a cleanup signal.
- File an upstream cliclack issue and link it from the comment.
- Add the manual repro recipe to the PR body (or to the issue) so future
  regressions have a known-bad state to test against.

## What I learned

This is the canonical "fix the race at the consumer boundary because the
library can't be fixed in time" pattern. The four-line patch with a
comment that names the race is exactly the right amount of code, and
exactly the right amount of documentation. The temptation to "do this
properly" with a poll loop or a synchronization channel is a trap — the
50ms `sleep` is ugly but it's also correct, debuggable, and removable.
The lifecycle work (named constant, TODO, upstream issue) is the
follow-up that turns a workaround into a tracked workaround, which is
the difference between technical debt and abandoned code.
