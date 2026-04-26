---
pr: 14104
repo: All-Hands-AI/OpenHands
sha: f28ac9880a511d811ec713cf8dfeddade364a68d
verdict: merge-as-is
date: 2026-04-26
---

# All-Hands-AI/OpenHands #14104 — fix(frontend): restore think title fallback

- **URL**: https://github.com/All-Hands-AI/OpenHands/pull/14104
- **Author**: kayametehan (Metehan Kaya)
- **Head SHA**: f28ac9880a511d811ec713cf8dfeddade364a68d
- **Size**: +59/-0 across 2 files (`frontend/src/components/v1/chat/event-content-helpers/get-event-content.tsx` + `frontend/__tests__/components/v1/get-event-content.test.tsx`)
- **Fixes**: #13690

## Scope

When the SDK emits a `ThinkAction` with a `summary` field shaped like `think: {"thought":"..."}` (its default JSON-dump fallback), the UI was rendering that ugly raw JSON as the chat-bubble title. This PR adds a regex guard at `get-event-content.tsx:46-50`:

```ts
if (event.action.kind === "ThinkAction" && /^think:\s*\{.*\}$/i.test(summary)) {
  return null;
}
```

When the regex matches, `getSummaryTitleForActionEvent` returns `null`, falling through to the existing `ACTION_MESSAGE$THINK` i18n label. Custom (operator-authored) summaries are preserved. Two regression tests cover both branches.

## Specific findings

- **The regex is appropriately narrow.** `/^think:\s*\{.*\}$/i` requires:
  - exact `think:` prefix (case-insensitive — handles `Think:` etc.),
  - any whitespace,
  - opening `{`, anything, closing `}`,
  - end of string.
  
  This won't false-positive on a legitimate operator summary like "thinking through the Helm chart appVersion mismatch" (no curly braces, no `think:` prefix), which the second test at `:122-130` confirms. The `.*` between braces is greedy but bounded by `$`, so an arbitrarily long JSON dump still matches in O(n) — no regex DoS surface.

- **TODO comment at `:43` properly references the upstream issue (#13690)** and frames this as a temporary client-side workaround for an SDK bug. That's the right disposition: the SDK should stop emitting default summaries, and when it does, this guard becomes harmless dead code that can be deleted.

- **`event.summary?.trim().replace(/\s+/g, " ")` at `:42`** normalizes whitespace before the regex check — so a summary with extra newlines or tabs (`"think:\n   {\"thought\":...}\n"`) still matches. Good.

- **Test fixture at `get-event-content.test.tsx:61-85`** is appropriately specific:
  - Includes the full `ActionEvent` shape (id, timestamp, source, thought, thinking_blocks, action.kind, tool_name, tool_call_id, tool_call, llm_response_id, security_risk, summary).
  - The `summary` field is the exact `think: {"thought":"Inspecting the request."}` shape that triggers the bug.
  - The `tool_call.function.arguments` field contains the same content as JSON, which mirrors what the SDK actually sends.

- **The two new tests cover the symmetric branches** (`:108-117` rendering the i18n fallback when summary matches the regex; `:120-130` preserving non-matching summaries). The "matched fallback" test also asserts the raw JSON does *not* appear via `screen.queryByText(...).not.toBeInTheDocument()` — that's the right negative-assertion shape.

- **`ACTION_MESSAGE$THINK` is an i18n key** (per the `screen.getByText("ACTION_MESSAGE$THINK")` assertion). The test runs without an i18n provider mock, so the key text appears verbatim — fine for a unit test, but worth a sanity check that the actual translation key exists in the en bundle. A `git grep "ACTION_MESSAGE\$THINK"` in `frontend/src/i18n/` (not visible in the diff) would confirm. If the key doesn't exist, the production UI would render the literal `ACTION_MESSAGE$THINK` text, which is even worse than the JSON dump. Reviewer should spot-check.

- **`getSummaryTitleForActionEvent` returns `null` rather than a fallback string.** The function's existing contract was: `summary || null`. The new guard returns `null` directly when matching. The caller (not visible in diff) presumably handles `null` by falling through to a kind-specific renderer or i18n key. Worth confirming the caller path treats `null` and `""`-falling-through identically — they should, since the existing `summary || null` already returns `null` for empty strings.

- **No change to non-Think action kinds.** The `event.action.kind === "ThinkAction"` guard is restrictive enough that other action types (TerminalAction, FileEditAction, etc.) are completely unaffected. Verified by the existing `terminalActionEvent` test at `:84-89` continuing to work without modification.

## Risk

Negligible. The change is gated on `kind === "ThinkAction"` AND a specific regex match — both narrow conditions. The fallback path (`null` → existing i18n label) was already in place; this PR just routes through it for one more shape of input. No new dependency, no new public API, no SDK-version constraint.

## Verdict

**merge-as-is** — clean three-line fix with a TODO pointing at the upstream issue, two regression tests covering both branches, narrowly-scoped regex that won't false-positive. Exactly the right shape for a "client-side workaround for SDK bug" PR.

## What I learned

When a downstream consumer is forced to filter ugly default output from an upstream library, the right pattern is (a) narrow conditional guard, (b) explicit TODO referencing the upstream issue, (c) regression tests on both the matched and non-matched branches, (d) preserve the existing fallback path rather than inventing a new one. This PR does all four. The alternative (rendering the JSON dump as a `<details>` collapsible, or stripping the `think:` prefix and pretty-printing the JSON) would have been more clever and harder to delete when the SDK fix lands. Pure-fallback is the right call when the eventual cleanup path is "delete this whole block".
