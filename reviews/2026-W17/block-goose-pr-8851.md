# block/goose #8851 — colorize context window indicator

- **Repo**: block/goose
- **PR**: #8851
- **Author**: angiejones (Angie Jones)
- **Head SHA**: 23cb4e9e47474fd3f868f101f4816d7d9830e350
- **Base**: main
- **Size**: +9 / −1 in `ui/desktop/src/components/ChatInput.tsx`.

## What it changes

Adds a `getContextAlertType(totalTokens, tokenLimit)` helper at
`ChatInput.tsx:61-67` that maps fill percentage to alert type:

```ts
if (percentage > 90) return AlertType.Error;
if (percentage > 75) return AlertType.Warning;
return AlertType.Info;
```

Then replaces the hard-coded `type: AlertType.Info` at line 555 with
`type: getContextAlertType(totalTokens || 0, tokenLimit)`.

## Strengths

- Real UX improvement: a static "Info" alert at 99% context is easy
  to miss and leads to surprise truncation/compaction. Tiered colors
  (Info → Warning → Error) follow standard UX conventions.
- Helper function is pure and self-contained — trivially
  unit-testable; could be moved to a shared utils file if other
  surfaces need the same mapping.
- `tokenLimit ? … : 0` guard avoids divide-by-zero when the limit
  isn't loaded yet. Combined with the call-site guard
  `isTokenLimitLoaded && tokenLimit`, this is doubly safe.
- `totalTokens || 0` defends against `undefined`/`null` from upstream.

## Concerns / asks

- The thresholds (75%, 90%) are hard-coded magic numbers. Two named
  constants (`CONTEXT_WARNING_THRESHOLD = 75`, `CONTEXT_ERROR_THRESHOLD = 90`)
  at the top of the file would document intent and make future
  tweaks reviewable.
- No tests. For a 9-line pure helper this is borderline, but a
  4-case unit test (50%, 80%, 95%, 0-limit) is essentially free
  insurance.
- The alert system likely caches the previous alert type; if the
  alert is re-`addAlert`'d every render with the same id but a
  changed type, behavior depends on `addAlert`'s upsert semantics.
  Worth verifying it actually re-renders on type transitions
  rather than ignoring the second call.
- The helper accepts `tokenLimit: number` but the `tokenLimit ? : 0`
  fallback suggests it may be `0` legitimately. Consider a typed
  `0`-or-positive contract or accept `tokenLimit | undefined`
  explicitly.

## Verdict

`merge-as-is` — small, safe, user-visible improvement. The
threshold constants and a 4-case test are both nice-to-haves but
nothing blocks merge.
