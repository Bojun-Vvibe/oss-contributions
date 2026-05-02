# Review — sst/opencode#25305

- PR: https://github.com/sst/opencode/pull/25305
- Title: fix(prompt): remove shell mode suspension from input traits
- Head SHA: `dfbda4f2a8fbffd762066b805ac25bc179d0d127`
- Size: +1 / −1 in `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx`
- Verdict: **merge-after-nits**

## Summary

`input.traits.suspend` was being set to `true` whenever the prompt entered
shell mode, which caused upstream consumers (key handlers / autocomplete
overlays) to behave as if the input were entirely disabled. The PR
narrows `suspend` to `!!props.disabled` and keeps `status: "SHELL"` as
the sole signal that shell mode is active.

## Evidence

```
-      suspend: !!props.disabled || store.mode === "shell",
+      suspend: !!props.disabled,
       status: store.mode === "shell" ? "SHELL" : undefined,
```
(line 568 of `prompt/index.tsx` at the head SHA above.)

## Notes / nits

- The change is semantically right — `status` already advertises the mode,
  so anything that needs to react to "we're in shell mode" can read that
  field instead of overloading `suspend`.
- No test coverage added. A small unit/snapshot that asserts
  `traits.suspend === false` and `traits.status === "SHELL"` while
  `store.mode === "shell"` would lock this behavior in; otherwise
  someone is going to re-introduce the OR-clause in three months when
  they want shell mode to also blank out completions again.
- Worth a quick scan of any handler that previously branched on
  `traits.suspend` while in shell mode — if there were any, they now
  need to switch to `traits.status === "SHELL"`. PR description doesn't
  mention auditing call sites; reviewer should grep before approving.

Tiny diff, low risk. Merge after either a regression test or a
confirmation that no consumer relied on the old `suspend === true`
behavior.
