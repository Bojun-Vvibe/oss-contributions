---
pr-url: https://github.com/QwenLM/qwen-code/pull/3609
sha: f771acb353ac
verdict: merge-after-nits
---

# fix(vscode-companion): slash command completion not triggering after message submit

Cross-package refactor lifting the `\u200B` (zero-width-space) handling out of four ad-hoc `text.replace(/\u200B/g, '').trim()` call sites and into a single shared utility `stripZeroWidthSpaces` (plus the named constant `ZERO_WIDTH_SPACE`) in `packages/webui/src/utils/inputPlaceholder.ts:1-25`, exported from `packages/webui/src/index.ts:263-266`. Then four consumers swap to the import: `packages/vscode-ide-companion/src/services/sessionExportService.ts:43`, `webview/handlers/SessionMessageHandler.ts:395`, `webview/hooks/useCompletionTrigger.ts:246`, `webview/hooks/useMessageSubmit.ts:53/96/107/195`, plus the `InputForm.tsx:206/341/348` triple at the source of the placeholder.

The load-bearing bug fix is at `useCompletionTrigger.ts:246-301` — the `handleInput` callback now reads `stripZeroWidthSpaces(inputElement.textContent || '')` *before* computing the cursor offset against the text. The prior version computed `cursorPosition` against the raw `textContent` (which still contained the zero-width-space) and then asserted `text.substring(0, effectiveCursorPosition)`, so when the input had a leading `\u200B` placeholder (set after a prior submit clears the field) the cursor offset was 1 ahead of the visible text length and the substring captured was wrong by one character — which silently broke the `/` trigger detection because the `lastIndexOf('/')` would land on a phantom position. The companion clamp at `:301-303` (`Math.min(... , text.length)`) defends against the residual case where the DOM cursor offset still exceeds the stripped length.

The new `useMessageSubmit.test.ts:1-131` is a properly scoped suite covering the stripping helper (single leading, multiple, mixed-with-other-whitespace, empty input) and the `shouldSendMessage` predicate across nine arms including `'\u200B'`-only, `'\u200B   '` (placeholder + whitespace), `'\u200Bhello'` (placeholder + real text), and the attachment carve-outs. The `parseExportSlashCommand` test additions at `sessionExportService.test.ts:131-139` pin the same contract for the export-command parser.

Three nits:

1. **`InputForm.tsx:341-348` re-computes `stripZeroWidthSpaces(inputText).trim().length === 0` twice** — once for the `data-empty` attribute and once for the `hasDraftContent` derivation at `:206`. A `useMemo`-cached `const trimmedText = stripZeroWidthSpaces(inputText).trim()` would shave a small amount of re-render work on every keystroke and make the predicate-vs-attribute coupling more explicit.
2. **The `ZERO_WIDTH_SPACE` constant is exported but only consumed inside the same package** (`useMessageSubmit.ts` writes it to `textContent` to maintain element height). Fine for now, but if downstream consumers start writing `\u200B` literals again the unification benefit erodes; a lint rule against `\u200B` literals outside `inputPlaceholder.ts` would lock the contract.
3. **`useCompletionTrigger.ts:298-303` clamp comment says "DOM cursor offset may exceed the stripped text length"** — true, but doesn't explain the *direction* of the off-by-one (placeholder character at position 0 shifts every offset by +1). A one-line example "e.g. raw `\u200Bfoo` has cursor at offset 4, stripped `foo` has length 3, so we clamp 4→3" would make the clamp's intent obvious to future readers.

## what I learned
Every `\u200B` (zero-width-space) placeholder in a contentEditable widget is a load-bearing invisible character that must be stripped at every read site. Centralising the strip-and-constant pair into one utility is the correct shape because it lets you grep for the *one* identifier rather than the literal `\u200B` character (which is invisible in most editors and often silently mangled by clipboards, terminal renderers, and code-review diff viewers). The companion gain is that the cursor-offset bug becomes visible the moment you trace the data flow — "raw textContent length includes the placeholder, stripped length doesn't, cursor offset was computed against the raw" — instead of being scattered across four call sites that each looked locally correct.
