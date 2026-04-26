# All-Hands-AI/OpenHands#14138 — refactor(frontend): consolidate loading states around shared spinner

- **Repo**: All-Hands-AI/OpenHands
- **PR**: [#14138](https://github.com/All-Hands-AI/OpenHands/pull/14138)
- **Head SHA**: `e1703340100c`
- **Author**: Scorbunny2
- **Base**: `main`
- **Size**: +125 / −123, 19 files

## Context

The frontend had at least four divergent spinner implementations: a
`<LoaderCircle>` from `lucide-react`, an inline
`animate-spin rounded-full ... border-t-2 border-b-2 border-primary` div,
the existing `<LoadingSpinner>` component, and a couple of one-off Tailwind
arrangements. This PR introduces a single `<Spinner>` component and migrates
~7 sites to use it.

## Change

1. **New** `frontend/src/components/shared/spinner.tsx` (57 lines) defines
   a `<Spinner size, label, className, spinnerClassName, labelClassName,
   testId />` component with four sizes (`sm`/`md`/`lg`/`xl` mapped to
   Tailwind `h-{4,5,12,16} w-* border-{2,2,4,4}`), a screen-reader-only
   `<span class="sr-only">{t(I18nKey.HOME$LOADING)}</span>` when no
   `label` is provided, and `role="status" aria-live="polite"` on the
   container — a real accessibility upgrade over the four originals.
2. **Migrations** (each ~2-line swap):
   - `agent-loading.tsx` — replaces `<LoaderCircle className="animate-spin
     w-4 h-4" color="white" />` with `<Spinner size="sm"
     className="text-white" testId="agent-loading-spinner" />`.
   - `hooks-loading-state.tsx`, `skills-loading-state.tsx`,
     `conversation-loading.tsx`, `branch-loading-state.tsx`,
     `repository-loading-state.tsx` — replace inline `<div
     className="animate-spin rounded-full h-8 w-8 border-t-2 border-b-2
     border-primary" />` blocks with `<Spinner size="lg" />`.
   - `modal-button-group.tsx` — replaces `<LoadingSpinner size="small"
     className="w-5 h-5" innerClassName="hidden" outerClassName="w-5
     h-5" />` (which already had a `hidden` inner — code smell from the
     old API) with `<Spinner size="md" className="text-white" />`.
   - `loading-spinner.tsx` (the existing primitive) is itself rewritten
     to delegate to `<Spinner>` rather than render its own SVG, keeping
     legacy callers working with `size="small"` → `size="md"`,
     `size="large"` → `size="lg"` shim. That's the right migration
     order — wrap before deleting.
3. **Tests**: new `frontend/__tests__/components/shared/spinner.test.tsx`
   (32 lines) verifies (a) `role="status"` and `sr-only` Loading text by
   default, (b) custom label rendering, (c) `size="xl"` produces the
   expected `h-16 w-16 border-4` classes.

## Strengths

- **Accessibility upgrade is the headline win**, not the dedup —
  `role="status" aria-live="polite"` plus an i18n'd `sr-only` label
  means every spinner is now screen-reader-friendly. The four originals
  had none of this.
- **Backward-compatible adapter**: rewriting `loading-spinner.tsx` to
  delegate (instead of deleting it outright) lets callers in other PRs
  in flight migrate at their own pace. Good migration hygiene.
- **`testId` prop pattern**: `data-testid={testId}` on the container
  preserves the existing `data-testid="agent-loading-spinner"` consumed
  by E2E tests. The diff shows the testid is preserved at every
  migration site — verified at `agent-loading.tsx:6`,
  `modal-button-group.tsx:48`.
- **Test for the size-class contract**: `expect(spinnerCircle).toHaveClass(
  "h-16", "w-16", "border-4")` locks the Tailwind class output so a
  future refactor can't silently break visual sizing. Same for the
  `sr-only` accessibility class.

## Risks / nits

1. **`color="white"` → `className="text-white"` swap is correct in
   theory, but `<Spinner>` uses `border-current` for the spinner ring,
   which inherits from `text-*`, not from `bg-*` or `border-*` directly.
   Verify visually that the dark-theme rendering still shows white;
   particularly the `agent-loading.tsx` site sits on a dark header so
   the regression surface is small but visible.
2. **`<LoaderCircle>` from `lucide-react` is also used in non-spinner
   contexts** (button "Loading…" affordances, file-upload chips). The
   PR scope correctly leaves those alone, but a follow-up should grep
   for any *spinner-intent* `LoaderCircle` callers that this PR missed
   and migrate them too. A quick `rg 'LoaderCircle.*animate-spin'`
   should give the remaining list.
3. **`SPINNER_SIZES.lg` is `h-12 w-12 border-4`** but the original
   inline div was `h-8 w-8 border-t-2 border-b-2`. That's a *visual
   change* — bigger ring, thicker border, full circle instead of just
   top/bottom arcs. Visually fine and probably an improvement, but
   worth a screenshot in the PR thread so reviewers don't think it's
   pixel-identical.
4. **No story / Storybook entry** for the new component (assuming the
   repo uses one). If not, ignore.
5. **`spinnerClassName` and `className` overlap risk**: a caller passing
   both might accidentally fight Tailwind specificity. Document the
   intent: `className` styles the container (gap, color, layout),
   `spinnerClassName` styles the rotating ring (border, size override).

## Verdict

**merge-after-nits** — the consolidation + a11y upgrade is well-scoped
and well-tested. Worth a screenshot pair (before/after for the `lg` size
sites) in the PR thread to confirm the visual change is intentional, plus
a one-line doc comment on the component clarifying `className` vs
`spinnerClassName`.

## What I learned

The pattern of "wrap the old primitive to delegate to the new one"
during a UI dedup migration is much safer than a flag-day rename. It
lets each call-site migrate as a separate PR while CI stays green
throughout. The only cost is the temporary double-import — easy to
remove once the wrapped form has zero callers.
