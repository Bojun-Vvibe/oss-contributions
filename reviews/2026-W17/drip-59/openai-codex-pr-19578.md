# openai/codex #19578 — fix: increase Bazel timeout to 45 minutes

- URL: https://github.com/openai/codex/pull/19578
- Head SHA: `f2d627f15eb8c43664d1673d390a0063fd9e1fc7`
- State: MERGED
- Files: `.github/workflows/bazel.yml` (+5/-1)
- Verdict: **merge-as-is**

## What it does

Bumps the Bazel job timeout from 30 → 45 minutes so cold-cache Windows
runs stop falling off the runner mid-build. Comment in
`.github/workflows/bazel.yml:18-22` explicitly frames this as a
stop-gap until distributed builds (BuildBuddy RBE) are wired in.

## Diff notes

The change is literally three lines: a triple-line comment plus
`timeout-minutes: 45`. From `bazel.yml:17-23`:

```
   test:
-    timeout-minutes: 30
+    # Ideally, this would be only 30 minutes, but a no-cache-hit Windows build
+    # seems to trip this limit and starting over is painful when it happens.
+    # Ultimately we need true distributed builds (e.g.,
+    # https://www.buildbuddy.io/docs/rbe-setup/) to speed things up.
+    timeout-minutes: 45
```

Nothing else in the workflow changes — same matrix, same triggers,
same concurrency group.

## Risk surface

- Minimal. The only effect of a longer timeout is that a genuinely
  hung job will hold a runner ~15 minutes longer before cancellation.
  Given the matrix already sets `cancel-in-progress: true` for
  non-`main` refs, queue depth on PRs is bounded.
- The comment correctly frames this as a temporary loosening rather
  than the permanent state of the world. Good signal for whoever picks
  up the BuildBuddy migration.

## Why this verdict

Pragmatic CI knob with an honest comment. No code paths affected, no
test changes needed. Lands clean.
