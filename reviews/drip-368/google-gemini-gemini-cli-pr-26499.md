# google-gemini/gemini-cli PR #26499 — fix: COPY from builder to runner

- Head SHA: `0252fe37a566a24c30dba9e5450d0e93bccad826`
- Author: `vital987`
- Size: +2 / -2
- Verdict: **merge-as-is**

## Summary

Two-line `Dockerfile` correctness fix at lines 84-85: the runner
stage was attempting to `COPY packages/cli/dist/google-gemini-cli-*.tgz`
and `packages/core/dist/google-gemini-cli-core-*.tgz` from the **build
context** rather than from the **builder stage**. In a clean
Docker build (no pre-built artifacts in the user's local checkout),
those `.tgz` files do not exist in the build context — they're
produced by the builder stage's `npm pack` (or equivalent) into
`/build/packages/{cli,core}/dist/`. The runner `COPY` therefore
either fails outright with `failed to compute cache key: …not
found` or, worse, silently picks up a stale `.tgz` from the user's
checkout if one exists locally, producing a runner image that
disagrees with the source tree the builder just compiled.

## What the diff actually does

```
-COPY --chown=node:node packages/cli/dist/google-gemini-cli-*.tgz /tmp/gemini-cli.tgz
-COPY --chown=node:node packages/core/dist/google-gemini-cli-core-*.tgz /tmp/gemini-core.tgz
+COPY --from=builder --chown=node:node /build/packages/cli/dist/google-gemini-cli-*.tgz /tmp/gemini-cli.tgz
+COPY --from=builder --chown=node:node /build/packages/core/dist/google-gemini-cli-core-*.tgz /tmp/gemini-core.tgz
```

The fix adds `--from=builder` (so `COPY` reads from the named
builder stage's filesystem, not the build context) and rewrites
the source path from `packages/…` (relative to context root) to
`/build/packages/…` (absolute path inside the builder stage,
which from the surrounding Dockerfile context is the builder's
WORKDIR convention). Destination paths and `--chown` are unchanged
so file ownership and downstream `npm install -g /tmp/*.tgz` at
the next `RUN` step are bit-identical.

## Why merge-as-is

- This is the canonical multi-stage Dockerfile fix pattern. The
  original was either typo-or-copy-paste from a single-stage
  build and is structurally wrong for a multi-stage layout.
- `--from=builder` is verifiable to be correct as long as a
  `FROM <something> AS builder` stage exists earlier in the
  Dockerfile *and* that stage produces the `.tgz` files at the
  named path — both of which are clearly the case (the diff
  context shows `npm install -g /tmp/gemini-core.tgz` followed
  by `JSON.parse(...)` smoke checks, which can only succeed if
  the `.tgz` files actually contain valid packages produced by a
  real build).
- Glob expansion (`*.tgz`) inside `COPY --from=<stage>` is
  supported by buildkit and will resolve against the builder
  stage's filesystem, so this preserves the version-suffix-
  agnostic shape.
- No tests are needed for a Dockerfile path correction; the
  smoke check at the next `RUN` step (parsing the installed
  package's `package.json`) functions as the regression test —
  if the new `COPY` paths are wrong, the smoke check fails the
  build immediately.
- Risk is genuinely zero: if the original was working in CI
  (i.e. the `.tgz` files happened to exist in the context), the
  fixed version still works because `--from=builder` reads from
  a strict superset of the same content. If the original was
  broken in CI, the fixed version unbreaks it. There's no path
  where this regresses.

## Optional nits (not blocking)

- The PR title `fix: COPY from builder to runner` is accurate
  but terse — `fix(Dockerfile): COPY tgz from builder stage,
  not build context` would be clearer in `git log`. Pure
  cosmetic.
- A 1-line commit-message body explaining *why* (i.e. "the
  builder stage produces these `.tgz` files via `npm pack`;
  reading from the context picked up stale artifacts or
  failed in clean builds") would aid future archaeology.
- Could fold a build-stage assertion (`RUN test -f
  /build/packages/cli/dist/google-gemini-cli-*.tgz` at the end
  of the builder stage) so a future regression in the builder
  produces an obvious "tgz not produced" error rather than a
  cryptic `COPY` failure later. Out of scope for this PR.
