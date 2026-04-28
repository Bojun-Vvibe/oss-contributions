# openai/codex#20051 — fix(tui): update plan mode task completion test

- **Repo:** [openai/codex](https://github.com/openai/codex)
- **PR:** [#20051](https://github.com/openai/codex/pull/20051)
- **Head SHA:** `480ef646aa3b09d4693322c4a5a8f5baab8ad37d`
- **Size:** +3 / -1 (one test file)
- **State:** OPEN

## Context

This is a follow-up cleanup to a build break in `upstream/main`. The author of
PR #20046 changed `on_task_complete`'s signature to take a new `duration_ms`
argument, but the existing test
`plan_mode_nudge_hides_while_task_or_modal_is_active` in
`codex-rs/tui/src/chatwidget/tests/plan_mode.rs:48` was still calling it with
the two-argument shape. Result: the `argument-comment-lint` job (and the test
build) fail on every PR until this lands.

The PR body links to the CI failures on #20046:

- [Argument comment lint - Linux job 73444711328](https://github.com/openai/codex/actions/runs/25069220827/job/73444711328)
- [Argument comment lint - macOS job 73444711416](https://github.com/openai/codex/actions/runs/25069220827/job/73444711416)

So the bug is real: trunk is red on this lint, and #20046 *can't* fix it
because #20046's `protocol.rs`-only diff doesn't touch `tui`. Splitting it
into a separate PR is correct — keeps the `protocol` PR scoped, lets this
land independently.

## Design analysis

The diff is the minimum possible:

```rust
// codex-rs/tui/src/chatwidget/tests/plan_mode.rs:48
-    chat.on_task_complete(/*last_agent_message*/ None, /*from_replay*/ false);
+    chat.on_task_complete(
+        /*last_agent_message*/ None, /*duration_ms*/ None, /*from_replay*/ false,
+    );
```

That matches the new three-arg signature in `chatwidget.rs` (PR body
confirms upstream/main already has the three-arg `on_task_complete`). The
`/*duration_ms*/ None` argument-comment is required by `just
argument-comment-lint` — codex's lint rule enforces inline doc-comments on
positional `Option`/`bool` arguments at call sites. The form here (`/*name*/
value`) matches the surrounding code style.

`None` is the right value semantically: "task completed, but we have no
recorded duration to report" — the test isn't exercising the duration code
path, it's exercising the nudge-visibility state machine, and `None` is the
neutral choice.

## Risks / nits

1. **Test is only a syntactic fix.** The author explicitly notes they
   couldn't run the test locally (`codex-linux-sandbox` won't build without
   `libcap`). That's fine for a 3-line argument-positional fix — the only way
   this breaks is if they got the argument *order* wrong. Reading
   `chatwidget.rs:on_task_complete` in trunk would confirm the `(message,
   duration, from_replay)` order; the PR body asserts that's the order on
   `upstream/main` and the comment values mirror it. Low risk.
2. **No assertion added for `duration_ms` semantics.** This PR is a build
   fix, not a coverage extension. Adding a separate test variant that passes
   `Some(Duration::from_millis(1234))` and asserts something downstream
   (e.g., that the metric line includes "1.2s") would be a valuable
   follow-up — but explicitly out of scope here.
3. **Race with #20046 landing.** If #20046 merges first, this PR is still
   needed (the test file lives in `tui/`, not `protocol/`). If this merges
   first, `chatwidget.rs` already has the three-arg signature on trunk so
   the test compiles correctly. Either order works.

## Verdict

**merge-as-is.** A 3-line trunk-unbreaking fix with the correct argument
comment style and a clear write-up of why it can't ride along with #20046.
No reason to delay.

## What I learned

- Codex's `argument-comment-lint` enforces `/*name*/ value` doc-comments on
  positional `Option`/`bool` arguments at call sites. That's a useful rule
  for keeping `on_task_complete(None, None, false)`-style calls readable —
  worth borrowing for any Rust codebase with similarly-shaped APIs.
- When a signature change lands in the same upstream commit as a callsite
  on a *different* file but only *some* callsites are updated, the resulting
  trunk break can survive review because each individual PR looks complete.
  Splitting the catch-up callsite fix into its own tiny PR (with links back
  to the failing CI jobs) is the right way to make the breakage and its
  resolution legible.
