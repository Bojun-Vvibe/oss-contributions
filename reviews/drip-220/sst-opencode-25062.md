---
pr-url: https://github.com/sst/opencode/pull/25062
sha: 9bddf7f3ef53
verdict: merge-as-is
---

# fix app crash restoring messages without model

One-character fix at `packages/app/src/context/local.tsx:385`: changes `msg.model.variant ?? null` to `msg.model?.variant ?? null` so the optional-chain short-circuits when restoring an old user message that lacks the `model` field (older session shape predates the per-message model field landing). Without the `?.`, the access throws `Cannot read properties of undefined (reading 'variant')` and the entire app remount on session restore crashes — taking out not just the single broken session but the in-process state of every other open session because the failure is in the top-level `LocalProvider` initialization path.

The fix is the right shape: don't try to migrate or coerce the legacy shape, just defend the read site so the resumed session degrades to "no variant" rather than crashing. The matching `model: msg.model` line one above stays as-is because `setSaved` accepts `undefined` there (the null vs undefined distinction matters for the variant column where `null` is the explicit "default" sentinel and `undefined` would round-trip to "field not set"). The PR body lists `bun typecheck` plus three colocated test files (`model-variant.test.ts`, `session-model-helpers.test.ts`, `submit.test.ts`) which is the right "I exercised the call paths that read `variant` after restore" coverage even without a new test pinning the regression — the fix is so narrow that a regression test would essentially assert "the optional-chain operator works."

The one missed-opportunity nit (small enough not to gate merge): the same file probably has other `msg.model.X` accesses in nearby code that will hit the same crash with the same legacy data — a quick `grep msg.model.` across the file would either confirm this is the only site or surface the next ticket.

## what I learned
"Crash on restore" is one of the worst UX bug classes because (1) the user can't recover by retrying — the bad data is now persistent, (2) it cascades — losing one session's state often wedges the whole app, and (3) it's invisible in dev because dev sessions all have the new shape. Defensive optional-chains at restore-from-storage boundaries are cheap insurance; the cost of one `?.` is ~zero, the cost of crashing every user with a 6-month-old session is "they uninstall."
