# QwenLM/qwen-code PR #3777 — fix(test): restore abort-and-lifecycle stdin-close test to pre-#3723 version

- PR: https://github.com/QwenLM/qwen-code/pull/3777
- Head SHA: `bf82093f3ba9e01e67fa00887cf9f79e8532d387`
- Author: @wenshao (Shaojin Wen)

## Summary

The E2E test `should handle control responses when stdin closes before replies` (in `integration-tests/sdk-typescript/abort-and-lifecycle.test.ts`) has been failing on every push to main since #3723 was merged, blocking the E2E Tests workflow on macOS and Linux for ~900s per run before timing out and retrying twice. The author's analysis is that #3723 included a "fix" commit that flipped the test's *semantics* (asserting `'updated'` instead of `'original content'`) so it now requires the CI LLM endpoint to deterministically emit a `write_file` tool call, accept the result, and reply `'done'` — which is not deterministic. This PR restores **only that test file** to the pre-#3723 version while keeping all of #3723's actual core changes (`permissionFlow.ts`, `coreToolScheduler.ts`/`Session.ts` rewiring, the `npm run bundle` step in the E2E workflow).

## Specific references from the diff

- `integration-tests/sdk-typescript/abort-and-lifecycle.test.ts:317-432` — replaces the post-#3723 `boundedPromise` harness (with phase-armed timers, `loopError` race, `secondResultMessage`, prompt `"Write \"updated\" to ${testFilePath}. Stop if any exception occurs."`, `canUseTool` filtering for `write_file && file_path === testFilePath`, and `expect(content).toBe('updated')`) with the simpler pre-#3723 form: a single `canUseToolCalled` flag + three resolver promises, prompt `"Use the write_file tool to write \"updated\" to the file at ${testFilePath}. Then reply with \"done\"."`, an unconditional `behavior: 'allow'` `canUseTool`, `q.endInput()` after `secondResultPromise`, and the load-bearing `expect(content).toBe('original content')` plus `expect(canUseToolCalled).toBe(true)`.
- The semantic flip (per PR table): canUseTool callback used to do `inputStreamDoneResolve()` → sleep 1s → resolve `allow`; the post-#3723 version resolved immediately. The asyncGenerator used to `await inputStreamDonePromise` → close stdin while a control reply was in flight; the post-#3723 version stayed open until `q.endInput()`. Final assertion used to be `'original content'` (in-flight tool not executed when stdin closes); post-#3723 was `'updated'` (full LLM tool-call round trip).
- Author cites pass on `npm run build`, `npm run bundle`, `tsc --noEmit`, `npm run lint`, and points at two prior failing runs (`4cd9f0cb` = #3723 itself, `8b6b0d64f` = #3771) and ~30 stable green runs prior to #3723 as evidence.

## Verdict: `merge-after-nits`

The empirical case is strong: the test went red on the exact commit that rewrote it, has been red on every subsequent push, and the rewritten assertion *requires* nondeterministic LLM behavior on the CI endpoint. Restoring the original semantics is the right call for now, since the test name (`stdin closes before replies`) actually describes what the pre-#3723 version exercises. The only thing keeping this off `merge-as-is` is that this is a revert-by-rewrite of a peer's deliberate change, not a bug fix in the conventional sense — it deserves explicit signoff from the author of #3723.

## Nits / concerns

1. **Tag the #3723 author for ack.** The PR body convincingly argues that #3723's test rewrite was a behavioral mistake, but this lands as a unilateral revert of someone else's test. Get an explicit ack from #3723's author (or at least a maintainer) on the PR before merge — otherwise the next time someone "fixes" this they'll re-introduce the broken version because they believe it's the intended semantics.
2. **Add a regression-protection comment.** Right above the new `expect(content).toBe('original content')` line, drop a `// IMPORTANT:` comment explaining "this assertion validates that an in-flight tool call is *not* executed when stdin closes mid-control-reply; do NOT change this to `'updated'` without re-reading PRs #3723 / #3777." Otherwise this trap will spring again.
3. **Track the unrelated macOS flake separately.** PR body mentions `interactive/cron-interactive.test.ts > loop fires inline in conversation` flaking on macOS since #3771 — explicitly file an issue and link it from the PR so it doesn't get bundled into the same revert conversation.
