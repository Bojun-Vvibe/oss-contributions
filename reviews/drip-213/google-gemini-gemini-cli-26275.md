# Review: google-gemini/gemini-cli#26275 — fix(hooks): preserve non-text parts in fromHookLLMRequest

- PR: https://github.com/google-gemini/gemini-cli/pull/26275
- Head SHA: `6fd57eeaa5321ff7fbf7782d6244b034ce899642`
- Fixes: #25558. Adapts implementation + tests from auto-closed PR #23340.
- Files: `packages/core/src/hooks/hookTranslator.test.ts` (+215/-0), `packages/core/src/hooks/hookTranslator.ts` (substantive — see below)
- Verdict: **merge-after-nits**

## What it does

Fixes an infinite tool-call loop triggered by any `BeforeModel` hook that mutates text
content in a conversation that already contains tool calls. `HookTranslatorGenAIv1.fromHookLLMRequest`
was rebuilding the SDK request's `contents` array text-only from `hookRequest.messages`,
silently discarding every non-text `Part` (`functionCall`, `functionResponse`, `inlineData`,
`thought`) from `baseRequest`. The model lost its tool-call history, didn't remember it had
already invoked the tool, and re-invoked it. After this PR, hook text edits are merged back
into `baseRequest.contents` *in place*, preserving non-text parts in their original positions.

## Notable changes

### Implementation (`hookTranslator.ts`)

- `:8-10` — `Content` and `Part` are added to the type imports from the GenAI SDK so the
  helper can narrow to the proper SDK types (was previously using ad-hoc `{ role: string; parts: unknown }`).
- `:104-110` — `isContentWithParts` is tightened from `content is { role: string; parts: unknown }`
  to `content is Content`. This is the load-bearing type-system change — it lets every downstream
  caller access `parts` as `Part[]` without `as` casts or eslint suppressions, which the PR body
  explicitly cites as a goal.
- (Body of `fromHookLLMRequest`, not in the captured diff but described in PR body) — walks
  `baseContents` in order, consuming hook messages by index. For entries that originally
  contributed text (i.e. produced a hook message), replaces text parts with the hook's edited
  text and preserves all non-text parts in their original positions. For entries that
  `toHookLLMRequest` skipped (no text parts — e.g. pure `functionResponse`), passes them
  through unmodified. Appends extra hook messages (new turns added by hook) at the end.
  Falls back to text-only when `baseRequest` is absent or has no contents (preserves the
  prior single-arg semantics, including the model-only-override path added in #22326).

### Tests (`hookTranslator.test.ts:174-388`)

Six new regression tests under a new `describe('fromHookLLMRequest with baseRequest (non-text part preservation)')`
block:

1. `:177-237` `should preserve functionCall parts when merging hook text back` — the canonical
   failure mode from the issue. Base has `[user-text, model-text+functionCall, user-functionResponse, model-text]`
   (4 entries); hook returns 3 messages (functionResponse skipped because text-only); asserts the
   merged result has 4 contents, with `functionCall` preserved at `contents[1].parts[1]` and
   `functionResponse` preserved as the only part of `contents[2]`.
2. `:243-292` `should handle text-only entries interleaved with function-only entries` — pins the
   skipped-entry index handling. Base is `[text, functionCall, functionResponse, text]`; hook
   returns 2 messages (the two function-only entries skipped); asserts the merged result preserves
   both function entries and updates both text entries.
3. `:294-330` `should collapse multiple text parts and preserve non-text parts` — pins the
   subtler case where one base entry has multiple text parts (`[text, text, functionCall]`) — the
   hook sees the concatenation and edits it; the merged result must collapse the two text parts
   into one (with the hook's edit) and preserve the trailing functionCall.
4. `:332-345` `should fall back to text-only when baseRequest is undefined` — pins that the
   single-arg call shape is byte-identical to the prior behaviour.
5. `:347-365` `should fall back to text-only when baseRequest has no contents` — pins the
   defensive case where `baseRequest` exists but `contents` is undefined.
6. `:367-388` `should append extra hook messages beyond base contents` — pins that hooks can
   *add* turns (not just edit existing ones); extra messages are appended at the tail.

## Reasoning

This is a real and load-bearing bug. The PR body cites the regulatory use cases (FERPA / HIPAA
PII redaction hooks) which is the obvious motivation, but the failure mode hits *any* hook that
touches text in a tool-using conversation — i.e. essentially every agent. Re-invoking the same
tool because the model forgot it already ran is exactly the kind of silent-loop failure that
burns tokens and confuses users.

The fix shape — merge in place rather than rebuild — is the right choice because the alternative
(extend `LLMRequest` to carry non-text parts through the hook surface) would be a breaking
change to the hook contract. Keeping the hook-facing API text-only and reconciling on the way
back preserves backward compatibility for every existing hook implementation.

The type-narrowing of `isContentWithParts` is good hygiene that the PR explicitly calls out;
removing the `as` casts and eslint suppressions reduces the surface for future drift.

The six tests are well-targeted: the canonical bug repro, the index-skipping case, the
multi-text-part collapse case, the two fallback cases, and the appended-hook-message case.
Together they pin the contract from every corner.

## Nits (non-blocking)

1. The PR body cites PR #23340 as the source of the implementation + tests but doesn't credit
   the original author in the commit metadata. If the project policy is to preserve original
   authorship via co-authored-by trailer, that should be added before merge. (If #23340 was
   genuinely auto-closed under a process policy, this is the right way to honour the
   contribution.)
2. The `describe` block name is long ("fromHookLLMRequest with baseRequest (non-text part
   preservation)"). Consider shortening to `fromHookLLMRequest: non-text part preservation`.
3. Test 3 (`:294-330`) asserts the collapsed text shape but the test stub passes the *already-
   concatenated* hook content `'I will search for you. [BLINDED]'`. A separate test where the
   hook *modifies* a multi-text-part entry to a different concatenated string would more
   strongly pin the "collapse on the way back" behaviour vs the trivial "single text part wins"
   case.
4. No test pins the case where the hook *removes* an entry (returns fewer messages than the
   base had non-empty text contents). The current implementation doc says hooks consume by
   index — what happens if hook returns 2 messages and base had 4 text entries? Either pin
   the behaviour explicitly (preserve trailing entries unchanged? drop them?) or document
   that hooks must not shrink the message list.
5. No test pins `inlineData` or `thought` part preservation, which the PR body explicitly
   names as part of the contract. The `functionCall` / `functionResponse` tests are good
   coverage but a one-line `inlineData` case would close the doc-vs-test gap.
6. Body cites `Gemini Code Assist's own review on that PR: "The implementation is robust...
   I found no issues."` — fine to include, but worth noting that AI-generated code review
   approval is not a substitute for human review; the fix here is mechanically right but
   the merge decision should rest on the human reviewer's read.
