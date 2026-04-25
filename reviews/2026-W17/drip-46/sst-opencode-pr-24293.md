# sst/opencode #24293 — fix(task): propagate parent session permissions to sub-agents

- **PR:** https://github.com/sst/opencode/pull/24293
- **Head SHA:** `1c166d0b4441d13e8ec6c69fa81db26a0f9a814d`
- **Files changed:** 2 — `packages/opencode/src/tool/task.ts` (+40/−28) and `packages/opencode/test/tool/task.test.ts` (+66)

## Summary

When the `task` tool spawns a sub-agent, the child session was previously created with a fresh permission array assembled only from sub-agent defaults (`todowrite: deny`, `task: deny`) and `experimental.primary_tools`. Any restriction on the parent — e.g. Plan-mode `edit: deny` — was silently dropped, letting the user bypass Plan mode by delegating the write through a sub-agent. This PR re-fetches the parent session and runs `Permission.merge(parent, childOverrides)` before calling `sessions.create`. Closes #6527.

## Line-level call-outs

- `packages/opencode/src/tool/task.ts:44-47` — the parent fetch is wrapped in `Effect.catchCause(() => Effect.succeed(undefined))`. Silently swallowing *every* cause is broad; it'll mask real failures (e.g. session-store IO errors) and degrade to "no inherited permission", which is the same fail-open mode the PR is trying to close. A narrower catch — only `SessionNotFound` or equivalent — would preserve the safety property: if we can't prove the parent has no restriction, refuse to spawn rather than spawning permissive. As written, a transient store error during sub-agent spawn re-opens the bypass.
- `:49-76` — call to `Permission.merge(parentSession?.permission ?? [], [...childOverrides])`. The merge ordering matters: in the test on `:140-152` the assertion is `findLast(rule => rule.permission === "edit")` returns `deny`, which only holds if `merge` appends parent-then-child and child overrides parent on duplicate keys. Worth a one-line code comment naming that contract — otherwise a future change to `Permission.merge`'s ordering silently flips this fix.
- `:77-82` — `sessions.create({ parentID, title, permission: childPermission })` no longer threads `cfg.experimental?.primary_tools`-style allow rules separately; they're now folded into `childPermission`. That's the right consolidation, but it does change how a primary-tools `allow` interacts with a parent's `deny` — under the new merge, a parent `edit: deny` will be overridden by a child `edit: allow` if `experimental.primary_tools` includes `edit`. That's probably intentional (primary tools are explicitly opted-in by the user), but it's a behaviour change worth calling out in the PR body or a CHANGELOG entry.
- `test/tool/task.test.ts:107-115` — sets parent to `edit: deny` via `sessions.setPermission`. Good. Missing: a negative-direction test that an `experimental.primary_tools = ["edit"]` config *does* override the parent deny (or, if that's not the intended behaviour, that it does not). The test only proves "parent restrictions survive default sub-agent spawn", not "the merge ordering is what we think it is".
- `test/tool/task.test.ts:140-152` — `findLast` rather than `find` for each rule type — correct given `Permission.merge` semantics, but again worth a one-liner comment so the next reader doesn't "fix" it to `find` and silently invert the test.

## Verdict

**merge-after-nits**

## Rationale

The bug is real and the fix is shaped correctly: re-derive child permission from `parent ∪ child-defaults` rather than `child-defaults` alone. The two nits worth landing before merge are (1) narrowing the parent-fetch catch so transient errors don't silently reopen the bypass, and (2) one explicit test that pins down the merge ordering between parent `deny` and child-side `allow` from `experimental.primary_tools`. Both are small. The unrelated whitespace churn in `provider.ts` that shows up in the same author's adjacent PR is not in this diff, so this one stays clean.
