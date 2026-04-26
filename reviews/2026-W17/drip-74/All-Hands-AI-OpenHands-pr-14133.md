---
pr: 14133
repo: All-Hands-AI/OpenHands
sha: 6f97594f17b67924268739ee7ebc5ecd1b32f2f4
verdict: merge-after-nits
date: 2026-04-26
---

# All-Hands-AI/OpenHands #14133 — feat(frontend): Add critic result types, component, and event rendering

- **Author**: xingyaoww
- **Head SHA**: 6f97594f17b67924268739ee7ebc5ecd1b32f2f4
- **Size**: +576/-14 across 11 files

## Scope

Adds CriticResult visualization to the GUI frontend. Critic events were already flowing through the SDK (per the PR body: "enabling the web UI to display critic evaluation results that are already flowing through SDK events"); this PR is the consumer-side render pass.

The pieces:
- New types in `frontend/src/types/v1/core/base/critic.ts:1-60` — `CriticResult { score, message, metadata }`, `CriticFeature { name, display_name, probability }`, `CriticCategorizedFeatures` (4 named buckets + `other`), `CriticMetadata` wrapping categorized features and event_ids.
- Optional `critic_result?: CriticResult | null` field added to `ActionEvent` (`events/action-event.ts:60-65`) and `MessageEvent` (`events/message-event.ts:18-22`).
- New display component `frontend/src/components/v1/chat/event-message-components/critic-result-display.tsx:1-408` — star rating (0-5 ★), score percentage, color-coded thresholds (green ≥60%, yellow ≥40%, red <40%), expandable feature breakdown.
- Renders inside `FinishEventMessage` (line 436) and `UserAssistantEventMessage` (line 485) when `event.critic_result != null`.
- 12 unit tests at `__tests__/.../critic-result-display.test.tsx`.

## Specific findings

- **Optional, additive, type-safe integration.** Both event types add `critic_result?: CriticResult | null` (lines 690, 713). Existing producers that don't emit critics see no shape change, and the consumer guards with `event.critic_result != null` (line 436, 485) — handles both `undefined` and `null`. This is the right shape for an opt-in renderer.

- **Render-only, no state mutation.** `CriticResultDisplay` (lines 357-408) takes `criticResult` as a prop, holds local `expanded` state (`React.useState(false)`, line 361), and renders. No Redux, no store wiring, no server round-trips. Means rollback is a one-line revert at each call site — exactly the right blast-radius for a new feature.

- **Threshold logic is in `getScoreColorClass`** (referenced at line 364). The PR body documents `green ≥60%, yellow ≥40%, red <40%`. Tests at lines 48-73 of the spec confirm: `0.8` → `text-green-400`, `0.5` → `text-yellow-400`, `0.2` → `text-red-400`. The thresholds are reasonable but **hard-coded magic numbers** — a future "configurable thresholds" issue is foreseeable. Don't block on it.

- **Star rating accessibility is the lurking nit.** Lines 382-384:
  ```tsx
  <span className={`${colorClass} text-xs tracking-wide`}>
    {"★".repeat(filled)}
    {"☆".repeat(empty)}
  </span>
  ```
  Screen readers will read this as "black star black star black star white star white star" — useless. Three of the tests (`renders 5 stars for a perfect score`, etc.) assert against the literal `"★★★★★"` text, so screen-reader users get the same meaningless text. Add `aria-label={t(I18nKey.CRITIC$RATING_LABEL, { filled, total: 5 })}` and `role="img"` so screen readers announce "rating: 4 out of 5" or similar. The percentage span next to it (`(72.0%)`, line 385) partially mitigates this but isn't grouped with the stars semantically.

- **Expand/collapse button has the right aria.** Lines 388-399 set `aria-label={expanded ? "Collapse details" : "Expand details"}` and `type="button"`. Tests confirm the label flips correctly. Good.

- **Categorized-features model has 4 named buckets + "other"** (`critic.ts:625-635`):
  - `agent_behavioral_issues`
  - `user_followup_patterns`
  - `infrastructure_issues`
  - `other`

  The bucket names commit to a specific opinion about *what* the critic categorizes. If the SDK ever produces a 5th category (e.g., `permissions_issues`, `cost_anomalies`), it'd land in `other` and the named-bucket renderer would silently drop it. The component already uses `categorized.other` correctly (line 342), so this isn't a present bug — it's a future-shape concern. Worth a comment in `critic.ts` saying "the four named buckets are derived from the SDK's CriticAnalyzer schema; new categories from the SDK will land in `other` until added here." Otherwise the next contributor will think the names are exhaustive.

- **`hasDetails` predicate is correctly defensive** (lines 368-373):
  ```ts
  const hasDetails =
    categorized != null &&
    ((categorized.agent_behavioral_issues ?? []).length > 0 ||
     (categorized.user_followup_patterns ?? []).length > 0 ||
     (categorized.infrastructure_issues ?? []).length > 0 ||
     (categorized.other ?? []).length > 0);
  ```
  Handles every undefined/empty path. The expand button only shows when there's content. Good.

- **`metadata` and `message` are typed as `string | null` / `CriticMetadata | null`** rather than `?`-optional (`critic.ts:653-660`). That's intentional — forces the SDK producer to always emit the field, even as `null`. Helps the renderer because there's no distinction between "field absent" and "explicitly null". Note: the *outer* `critic_result?: CriticResult | null` on the event types still allows the field to be absent at the event level. Slight asymmetry but it's the right one.

- **i18n is wired up** for the label and the categorized-feature labels (PR body claims "15 languages"). The test at line 81 asserts the literal i18n key `CRITIC$SUCCESS_LIKELIHOOD_LABEL` is rendered, which means in the test environment the i18n stub returns the key. Real-language coverage will be confirmed by the i18n-key-coverage check on CI; if that check doesn't exist for this repo, manually grep for `CRITIC$` in `i18n/translation.json` (file is in the diff, line ~7 of the file list).

- **Nit: 12 tests is the right shape.** Score percentage, full/zero stars, color thresholds (3), label rendering, expand-button presence/absence (with and without features), expand-on-click (1), expanded content per category. The one missing is "renders nothing / no extra space when `criticResult` is null" — but `CriticResultDisplay` requires `criticResult: CriticResult` as a non-null prop, and the gate is at the call sites (`FinishEventMessage` line 436, `UserAssistantEventMessage` line 485). So the null-handling test belongs at the *call-site* component level, not on `CriticResultDisplay` itself. Add one test in `finish-event-message.test.tsx` (if it exists) asserting that no `CriticResultDisplay` is mounted when `event.critic_result == null`.

- **`UserAssistantEventMessage` gates on agent source** (line 485):
  ```tsx
  {event.source === "agent" && event.critic_result != null && (
    <CriticResultDisplay criticResult={event.critic_result} />
  )}
  ```
  Good — critic results from user-source events would be confusing UX. `FinishEventMessage` (line 436) doesn't gate on source because finish actions are unambiguously agent-side. Both right.

- **No mention of dark/light theme.** Color classes (`text-green-400`, `text-yellow-400`, `text-red-400`, `text-neutral-300/400/500/600`) are Tailwind tokens that work for the dark theme; the contrast on light theme should be checked. If OpenHands' frontend is dark-only, ignore.

- **Nit: percentage display precision.** `(criticResult.score * 100).toFixed(1)` (line 365) — so `0.72` renders as `(72.0%)`. The trailing `.0` looks deliberate (consistent width) but reads slightly odd. Either keep `.toFixed(1)` (current) or use `.toFixed(0)` for "(72%)". Reviewer's call; not worth blocking.

## Risk

Low. Strictly additive — new optional fields, new render component, new translations. Existing event flows unaffected. Worst case the threshold colors clash with the chosen theme or i18n keys are missing for some language, both of which are visible-and-fixable bugs, not silent corruption.

The accessibility nit (star screen-reader text) is the only finding I'd hard-block on for an enterprise audit but soft-block for community review — it's a real bug for blind users today, but the percentage span partially mitigates it and the fix is one prop.

## Verdict

**merge-after-nits** — clean additive feature with the right test density and right opt-in shape. Three soft nits:
1. Add `aria-label` + `role="img"` to the star-rating `<span>` so screen readers announce "rating: N out of 5" instead of "black star white star..."
2. Add a one-line comment in `critic.ts` next to the four named buckets noting that new SDK categories will land in `other` until explicitly added here.
3. Add a call-site test in the `FinishEventMessage` / `UserAssistantEventMessage` specs asserting `CriticResultDisplay` is *not* mounted when `event.critic_result == null`.

## What I learned

The "screen reader reads `★★★★☆` as 'black star black star...'" failure mode is one of the easiest accessibility regressions to ship and one of the hardest to detect in code review without explicit a11y discipline. Tests that assert on the literal star characters reinforce the bug rather than catching it — the test passes precisely because the bad behavior is the asserted behavior. The fix is one prop, but spotting it requires a reviewer who's used a screen reader, or an axe-core CI check. Worth adding to OpenHands' frontend test infrastructure if it's not there.
