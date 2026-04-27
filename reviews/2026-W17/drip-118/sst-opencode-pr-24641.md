# PR #24641 — fix(app): refresh workspace dirs before canOpen check in navigateToProject

- **Repo**: sst/opencode
- **PR**: #24641
- **Head SHA**: `c1639f48`
- **Author**: HaleTom (Tom Hale)
- **Size**: +2 / -0 across 1 file
- **Verdict**: **merge-after-nits**

## Summary

Two-line fix inside `openSession()` (and the SDK-resolution branch
right below it) in `packages/app/src/pages/layout.tsx:1295-1311`
that calls `await refreshDirs(target.directory)` before the
`canOpen(target.directory)` check. The bug shape is a real race
on Windows CI: the test enables workspaces and then immediately
tries to switch back to a project, but `project.sandboxes` is
populated asynchronously by `worktree.list` + `fsmonitor`, so the
workspace directory is not yet in the cached `dirs` set when
`navigateToProject → openSession` runs. `canOpen` returns false,
`openSession` bails silently, and the function falls through to
navigate to the wrong URL — which is exactly what `toHaveURL`
catches in `projects-switch.spec.ts:90`. Author also threads the
same `refreshDirs(resolved.directory)` into the SDK
session-resolution path because a resolved session can land in a
workspace dir that hasn't been registered yet either.

## Specific changes

- `packages/app/src/pages/layout.tsx:1298` — `await refreshDirs(target.directory)` before the existing `if (!canOpen(target.directory)) return false`. `refreshDirs` is documented to fetch the worktree list from the SDK only when the cached `dirs` set doesn't already contain the target, so the cost is bounded (no extra round-trip on the steady-state path).
- `packages/app/src/pages/layout.tsx:1310-1311` — same `await refreshDirs(resolved.directory)` immediately after the SDK `session.get(...)` resolves and before the second `canOpen` check.

## Risks

- **No regression test**: the PR description names a specific failing e2e (`projects-switch.spec.ts:90`) but the diff doesn't make that test deterministic — it just removes the race window enough that the existing flake stops firing. A targeted unit test that constructs a workspace dir not yet present in the cached `dirs` set, calls `openSession`, and asserts `refreshDirs` was awaited would pin this against re-introduction. Without it, the next refactor of `navigateToProject` can silently drop the await.
- **Sequential awaits on a hot path**: this turns the cheap `canOpen` shortcut into a guaranteed network round-trip (when `target.directory` isn't cached) for every sidebar project click. The author argues `refreshDirs` no-ops when the dir is already in `dirs` — worth a one-line comment at the call site naming that invariant so a future reader doesn't "optimize" the await away. If the no-op assumption ever breaks (e.g. someone changes `refreshDirs` to always fetch), every project switch eats a round-trip.
- **Two awaits, two failure modes**: if `refreshDirs` itself rejects (network blip, SDK down), `openSession` now propagates that rejection to the caller instead of falling through to `canOpen→false→return`. The current call site doesn't wrap either await in `.catch(() => {})`, so a transient failure becomes a user-visible error rather than a silent "can't open this project right now". Probably right (loud > silent for sandbox state), but worth confirming with maintainer.

## Verdict

`merge-after-nits` — the race is real, the diagnosis (closes #22051,
#21172, #14461 — three flake reports against the same test) is
credible, and the fix is the smallest possible thing that closes
the window. Two-line review asks: (1) add a regression test that
constructs the "workspace dir not in cached `dirs`" precondition
and asserts `openSession` waits for `refreshDirs` before
`canOpen`, and (2) a one-line comment naming the no-op invariant
that makes the extra await acceptable on the steady-state path.

## What I learned

The pattern of "cached predicate + lazy refresh-on-miss" is
deceptively common in front-ends that mirror server state, and
the failure mode is always the same: a writer enables some new
state and an immediately-following reader checks the predicate
before the cache catches up. The right shape is what this PR
does — refresh before the read whenever a freshly-enabled state
might be in play — but the cost is paid by every steady-state
read too. The middle ground that doesn't show up in this diff
is to invalidate the cache on the writer side (when workspaces
get enabled, mark `dirs` stale and trigger one refresh) so the
reader can stay synchronous. That's a bigger change than this
PR's scope, but worth filing as a follow-up if the
refresh-on-every-click latency becomes visible.
