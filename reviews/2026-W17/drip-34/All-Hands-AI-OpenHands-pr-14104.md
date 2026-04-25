# All-Hands-AI/OpenHands #14104 — Restore think title fallback

- **Repo**: All-Hands-AI/OpenHands
- **PR**: [#14104](https://github.com/All-Hands-AI/OpenHands/pull/14104)
- **Head SHA**: `f28ac9880a511d811ec713cf8dfeddade364a68d`
- **Author**: kayametehan
- **State**: OPEN
- **Verdict**: `merge-as-is`

## Context

The frontend's `getEventContent` helper renders an action title for every
agent event in the chat timeline. For `ThinkAction` events the SDK
currently emits a default `summary` of the form
`think: {"thought":"..."}` — i.e., the JSON-serialized arguments. That
string is then used verbatim as the row title, which is both noisy and
leaks tool-call internals into the user-visible UI. The fallback path
that would substitute a localized `ACTION_MESSAGE$THINK` label was
unintentionally bypassed because the summary string was always
non-empty.

## Design

Two surgical changes:

1. **`get-event-content.tsx:44-50`** adds a single guard before the
   summary is returned:

   ```tsx
   if (
     event.action.kind === "ThinkAction" &&
     /^think:\s*\{.*\}$/i.test(summary)
   ) {
     return null;
   }
   ```

   Returning `null` from `getSummaryTitleForActionEvent` is the existing
   contract for "let the upstream caller pick the localized fallback."
   The TODO comment correctly references upstream issue #13690 as the
   source-of-truth for retiring the workaround once the SDK stops emitting
   the default summary.

2. **`get-event-content.test.tsx:61-131`** adds a `thinkActionEvent`
   fixture and two test cases: one asserting the fallback fires for the
   default JSON-dump summary (`screen.getByText("ACTION_MESSAGE$THINK")`)
   and one asserting that a non-default summary
   ("Thinking through the Helm chart appVersion mismatch") is preserved
   verbatim. Both tests also `queryByText` the negative case to lock the
   exclusivity. Solid pair.

The regex `/^think:\s*\{.*\}$/i` is appropriately narrow — it matches
exactly the SDK's emit format (`think:` literal + brace-delimited JSON
trailer) and does not accidentally match user-authored summaries that
happen to start with the word "think". The `i` flag is defensive.

## Risks / Nits

1. **`.*` is greedy across newlines.** In JS without the `s` flag, `.`
   does not match newlines, which is what you want here — multi-line
   user summaries that happen to start with `think: {` and end with `}`
   on a different line will not be treated as "default". The current
   regex is correct but a one-line comment ("greedy intentional, single
   line only") would prevent a future contributor from "fixing" it to
   `[\s\S]*`.

2. **SDK contract.** The TODO references issue #13690, which is the
   right tracking link, but the TODO has no automatic enforcement
   (e.g., a SDK-version check). When #13690 lands, this guard becomes a
   no-op rather than dead code — which is fine, but consider a
   one-liner test comment cross-referencing the SDK fix so the fixture
   gets removed at the same time.

3. **Localization key check.** `ACTION_MESSAGE$THINK` is asserted as a
   raw key in the test, which presumes the test environment doesn't
   resolve i18n. Confirm that's intentional vs. wanting the test to
   pull from the actual translation table — both are reasonable, but
   the mismatch can confuse future readers.

## What I learned

The pattern here — "SDK ships a placeholder default that the UI then
needs to detect-and-suppress" — is a recurring smell when two layers
disagree about who owns the empty/default state. The cleaner long-term
fix is the linked SDK issue (don't emit a default summary at all, let
the consumer decide); the short-term suppression here is the right
shape because it's narrowly scoped, documented with a tracking link,
and accompanied by both positive and negative tests so its eventual
removal is mechanical.
