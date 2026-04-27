# sst/opencode#24621 — fix(app): auto-add project to sidebar when navigating via direct URL

- PR: https://github.com/sst/opencode/pull/24621
- Head SHA: `cceba3e6`
- Diff: +14 / -1 in `packages/app/src/pages/directory-layout.tsx`
- Closes #18943, #16837

## What it does
Adds a `createEffect` in `directory-layout.tsx:58-66` that, whenever `resolved()` returns a non-empty directory, calls `layout.projects.open(dir)` and `server.projects.touch(dir)` inside `untrack(...)`. This fixes the case where someone lands on `/{base64(dir)}/session` directly (IDE plugin, deep link, page reload) and the sidebar never picks up the project because `layout.projects.open` was only ever called from the home flow.

## Strengths
- Author correctly relies on the documented idempotency of `layout.projects.open()` (no duplicate sidebar entries on repeat visits) and pairs it with `server.projects.touch(dir)` so the recents/last-opened ordering also updates — this is what the home-screen flow does.
- `untrack(...)` is the right call: without it, anything reactive read inside `open`/`touch` would re-trigger the effect and you'd have a thrash loop. Good defensive choice.

## Concerns
- The effect tracks `resolved()` but not `params.dir` directly. `resolved` is a `createMemo` over `params.dir`, so this is fine — but a one-line comment would help future readers who see two `createEffect`s back-to-back (`directory-layout.tsx:58` and the existing one at `:67`) both keying off the same param.
- No error handling around `server.projects.touch(dir)`. If the sync server is down on a fresh page load, this throws into the effect and SolidJS will surface it as an unhandled error. Wrapping in `.catch(() => {})` (or a debug log) matches the tone of "best-effort sidebar hint."
- No test added. The PR description lists manual reproduction steps but the file has no companion `*.test.ts(x)`. Given the bug has shipped twice (two closing issues), a small unit test against the sidebar store would be cheap insurance.

## Verdict
**merge-after-nits** — fix is correct and small; encourage author to add a regression test and harden the `touch` call before merge, but neither is a blocker for the bug fix itself.
