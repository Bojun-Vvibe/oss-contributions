# QwenLM/qwen-code #3737 — preserve reasoning_content in rewind, compression, merge paths

- **PR:** https://github.com/QwenLM/qwen-code/pull/3737
- **Title:** `fix(core): preserve reasoning_content in rewind, compression, and merge paths (#3579)`
- **Author:** fyc09
- **Head SHA:** 480f9e71b4c4af525aeb5c591f3f078da7369233
- **Files changed:** 8, +48 / −363 (net deletion — most of the diff is
  removing `stripThoughtsFromHistory()` and its call sites/tests)
- **Verdict:** `merge-as-is`

## What it does

Closes the three remaining holes from #3579 where DeepSeek's required
`reasoning_content` continuity could be broken mid-session. The
canonical symptom: a DeepSeek thinking-mode session that survives a
`/rewind` or autocompression suddenly starts returning 400-class errors
because the API saw an assistant turn without its preceding
`thought` part.

The three paths fixed in this PR (per the PR body, with the visible diff
in `AppContainer.tsx:1740-1745`):

1. **Rewind** — `AppContainer.tsx` previously called
   `geminiClient.stripThoughtsFromHistory()` after truncating
   history. The diff removes the call and replaces the comment to
   explain *why*:

   ```ts
   // 3. Truncate API history to the target point.
   // Do NOT strip thought parts — reasoning models (e.g. DeepSeek)
   // require reasoning_content continuity across all turns in the
   // conversation.
   geminiClient.truncateHistory(apiTruncateIndex);
   // (stripThoughtsFromHistory call removed)
   ```

2. **Chat compression** — the compressed summary turn used to be
   inserted without a `thought` part, breaking the chain.
3. **Message merge** — `mergeConsecutiveAssistantMessages` in
   `converter.ts` used to drop `reasoning_content` when merging
   adjacent assistant messages.

The second commit then deletes `stripThoughtsFromHistory()` and
`stripThoughtsFromHistoryKeepRecent()` (now dead code) and prunes their
mocks/assertions across `config.test.ts`, `client.test.ts`, etc. — that's
where the bulk of the −363 deletions come from.

## What's good

- The fix is *deletions of incorrect behavior*, not new logic. That is
  the right shape for this kind of bug: the original
  `stripThoughtsFromHistory` call sites were defensive cleanup that
  predates the DeepSeek/reasoning-model integration; once the system
  treats `thought` parts as load-bearing, stripping them anywhere in
  the lifecycle is wrong, full stop.
- Removing the function entirely (rather than leaving it dormant) is
  the right call — it eliminates the chance that a future contributor
  re-introduces a strip call somewhere new.
- Test surgery is consistent: every place that mocked
  `stripThoughtsFromHistory: vi.fn()` is removed in the same PR, e.g.
  `config.test.ts:171`, `config.test.ts:479-481` (`expect ... not.toHaveBeenCalled` is gone),
  `client.test.ts:474, 514, 533, 1225, 1279`. No orphaned mocks.
- The rewind comment (`AppContainer.tsx:1742-1744`) explains the
  *why*, names the affected provider class ("reasoning models, e.g.
  DeepSeek"), and describes the invariant
  ("reasoning_content continuity across all turns"). Future readers
  who see only the removed call will understand why it was removed.
- Good linkage hygiene: PR body cites #3579 (umbrella issue), #3590
  (session resume / idle fix), and #3682 (model switch / load_history
  fix) so the full lineage is reconstructable from a single PR.

## Nits / risks

- I cannot see the `chatCompressionService.ts` and `converter.ts`
  diffs from the head of the PR diff (the `gh pr diff` output I
  reviewed was truncated to the rewind site and the test deletions).
  The PR body promises both fixes; reviewer should confirm visually
  that:
  - the compressed-summary turn now carries a `thought` part (or that
    the compression path is now entirely `thought`-aware), and
  - `mergeConsecutiveAssistantMessages` concatenates
    `reasoning_content` rather than dropping it on merge.
- No new test that asserts a DeepSeek session survives `/rewind` end-
  to-end. Each of the three sub-fixes likely has its own unit test
  in this PR, but a single integration test that does
  `start session → think → tool call → /rewind → think again →
  succeed` would lock in the invariant against future regressions of
  any of the three paths.
- `stripThoughtsFromHistoryKeepRecent` is also being deleted (per the
  test diff). Confirm there's no remaining caller — a quick `grep` on
  the merged tree at HEAD should show zero hits. From the test
  pruning pattern it looks clean, but worth a final scan.
- Net diff is large in absolute lines but the vast majority is
  mechanical deletion of mocks. The reviewer should focus on the
  three logic sites (rewind, compression, merge) and skim the test
  deletions.

## Verdict rationale

`merge-as-is`: this is the kind of fix where doing nothing additional
is the right answer. The behavior being removed was the bug; the new
comment explains why; the dead function is gone so the bug can't be
reintroduced trivially. Worth merging once a maintainer has eyeballed
the compression and merge sites that I couldn't see in the truncated
diff.
