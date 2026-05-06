# Review: google-gemini/gemini-cli#26542

- **PR:** [fix(core): allow redirection in YOLO and AUTO_EDIT modes without sandboxing](https://github.com/google-gemini/gemini-cli/pull/26542)
- **Head SHA:** `ce05d74004690ef243ec99fcad4ffdd1e73d2aeb`
- **Merged:** 2026-05-05T21:37:15Z
- **Verdict:** `needs-discussion`

## Summary

Drops the `sandboxEnabled` precondition from `PolicyEngine.shouldDowngradeForRedirection`, so commands containing redirection (`|`, `>`, `2>&1`, etc.) are no longer downgraded to `ASK_USER` in `YOLO` and `AUTO_EDIT` modes when no sandbox is active. Adds one regression test asserting the new behavior.

## Specific notes

- `packages/core/src/policy/policy-engine.ts:288-296` — the conditional was previously `sandboxEnabled && (mode == AUTO_EDIT || mode == YOLO)`, now just `(mode == AUTO_EDIT || mode == YOLO)`. The `sandboxEnabled` local is removed entirely. New comment claims "These modes trust the agent's actions (YOLO) or specific task (AUTO_EDIT)."
- `packages/core/src/policy/policy-engine.test.ts:1900-1922` — single new test covers YOLO + `NoopSandboxManager` + a piped redirection command, asserts ALLOW. There is no matching test for `AUTO_EDIT` despite the change applying equally to it.
- The original code's intent was almost certainly a defense-in-depth measure: redirection broadens the blast radius of a shell command (file overwrite, `2>&1` to a sensitive path, `| sh`-style chains), and the sandbox was the safety net that justified bypassing the user prompt. Removing it means YOLO/AUTO_EDIT users on a non-sandboxed host get *less* protection than before this PR for an entire class of risky commands.

## Rationale

The behavior change is straightforward and the test passes, but the security posture trade-off deserves a maintainer call before merging:

1. **YOLO without a sandbox is now strictly more permissive.** That may be the intended product direction (YOLO = "I accept the risk"), but the previous code authors clearly disagreed since they hard-gated this on `sandboxEnabled`. The PR description frames it as a "regression," but the git blame on the `sandboxEnabled` check would clarify whether it was added intentionally.
2. **AUTO_EDIT is conceptually different from YOLO.** AUTO_EDIT typically signals "trust file edits, but still confirm shell side-effects." Allowing arbitrary redirected shell commands without prompt and without sandbox in AUTO_EDIT is a meaningful expansion of trust — it should at minimum be called out separately in the PR description and have its own regression test.
3. **No `AUTO_EDIT` test.** The new test only covers `YOLO`. Equivalent `AUTO_EDIT` coverage is needed before this can land safely; otherwise a future refactor could quietly re-tighten one mode and not the other and no test would catch it.
4. The replaced inline comment is generic ("These modes trust the agent's actions"). If the maintainers do want this behavior, the comment should explicitly acknowledge the dropped sandbox requirement so the next reviewer understands why an apparent safety check was removed.

Recommend a maintainer-level discussion on (1) and (2), an `AUTO_EDIT` test for (3), and a more explicit comment for (4) before merging. The mechanical change itself is small and easy to reverse if the call goes the other way.
