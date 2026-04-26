---
pr: 3640
repo: QwenLM/qwen-code
sha: cc9e65365d61130ee70ae731bd29059cd9d6453e
verdict: merge-as-is
date: 2026-04-26
---

# QwenLM/qwen-code #3640 — fix(cli): guard gradient rendering without colors

- **URL**: https://github.com/QwenLM/qwen-code/pull/3640
- **Author**: yiliang114 (易良)
- **Head SHA**: cc9e65365d61130ee70ae731bd29059cd9d6453e
- **Size**: +98/-9 across 5 files (Header + StatsDisplay + new gradientUtils + tests)

## Scope

Adds a guard around `<Gradient>` rendering so the TUI degrades to plain text instead of crashing with `Invalid number of stops` when:

1. `NO_COLOR` is set (the standard env-var convention) and `theme.ui.gradient` resolves to colors that ink-gradient can't interpolate, or
2. A custom theme defines `ui.gradient` with fewer than 2 colors (single-stop gradient is invalid), or
3. Gradient colors are an empty/whitespace string.

The actual fix is a new utility:

```ts
// packages/cli/src/ui/utils/gradientUtils.ts (new, +18 lines)
export function getRenderableGradientColors(
  ...candidates: Array<string[] | undefined>
): string[] | undefined {
  return candidates.find(
    (colors): colors is string[] =>
      Array.isArray(colors) &&
      colors.length >= 2 &&
      colors.every(
        (color) => typeof color === 'string' && color.trim().length > 0,
      ),
  );
}
```

Then `Header.tsx:102-109` and `StatsDisplay.tsx:198-201` call it, and conditionally render the `<Gradient>` wrapper or a plain `<Text>`:

```tsx
// Header.tsx
{gradientColors ? (
  <Gradient colors={gradientColors}>
    <Text>{displayLogo}</Text>
  </Gradient>
) : (
  <Text>{displayLogo}</Text>
)}
```

## Specific findings

- **The variadic-candidates API is the right shape.** `getRenderableGradientColors(theme.ui.gradient, [theme.text.secondary, theme.text.link, theme.text.accent])` walks the candidate list in order and returns the first one that meets `length >= 2 && every color is non-empty trimmed string`. This replaces the previous `theme.ui.gradient || [fallback]` pattern at `Header.tsx:101-105` (pre-diff), which was buggy because a *defined-but-invalid* `theme.ui.gradient` (e.g. `['red']` from a user's custom theme) would short-circuit the fallback and crash inside ink-gradient. New code falls through to the next candidate when the first is unusable. Good design.

- **`length >= 2` is the correct threshold.** ink-gradient's `gradient-string` underneath needs at least 2 stops to interpolate between. The thrown error `Invalid number of stops` (cited in `StatsDisplay.test.tsx:600` assertion) confirms this is the exact failure mode that was being hit.

- **`color.trim().length > 0` defends against accidental empty strings** in a custom theme JSON. Worth pairing with a follow-up theme-validation step that rejects this at config load instead of silently fall-through, but as a runtime safety net it's correct.

- **Test coverage is solid.**
  - `Header.test.tsx:96-103` — sets `NO_COLOR=1` and asserts the ASCII logo still renders (i.e. doesn't crash), with `beforeEach`/`afterEach` that save/restore the original env.
  - `StatsDisplay.test.tsx:578-603` — registers a `OneColorGradient` custom theme via `themeManager.loadCustomThemes`, activates it, and asserts the rendered output contains the title text and *not* the `Invalid number of stops` error string. This is the exact regression the fix targets.
  - The `themeManager.loadCustomThemes({})` + `themeManager.setActiveTheme(DEFAULT_THEME.name)` reset in `afterEach` (StatsDisplay test, lines 47-53) is important hygiene — without it a test that activates `OneColorGradient` would leak into subsequent tests in the same file.

- **`NO_COLOR` env-var handling.** This PR doesn't *directly* read `NO_COLOR` — it relies on whatever upstream resolves `theme.ui.gradient` to nil/empty when `NO_COLOR` is set. The test at `Header.test.tsx:96` exercises that integration. Worth a follow-up to confirm the theme resolver actually honors `NO_COLOR` (the test asserts behavior, not the resolver's logic). If it doesn't, the `NO_COLOR` test passes for the wrong reason. Not a blocker for *this* PR though — its job is "don't crash regardless of why gradient is invalid."

- **Both call sites consolidated to one helper.** `Header.tsx` and `StatsDisplay.tsx` both used to inline their own gradient resolution; both now go through `getRenderableGradientColors`. If/when a third gradient-using component lands (e.g. a fancy banner in another screen), this is the obvious utility to reach for. License header on the new file matches the project convention.

- **No public API change.** The new utility is internal to `packages/cli`; no exports added to a published surface.

## Risk

Negligible. Pure defensive guard. The only behavior change is "previously crashed with `Invalid number of stops` → now renders plain text." If anyone was relying on the crash as a debugging signal (they aren't — there's no test that asserts the throw), they'd notice. Otherwise this is strictly safer.

## Verdict

**merge-as-is** — well-scoped fix, clean utility extraction, integration tests at both component sites, env-var save/restore hygiene in test setup, license header on new file. Nothing to nit.

## What I learned

The "fall-through chain of candidate configurations" pattern (`getRenderableGradientColors(...candidates)`) is a much better shape than the more common `value || fallback` because it survives a defined-but-invalid value. The same pattern applies to any rendering call where a downstream library throws on bad-but-truthy input — log levels, color names, font sizes, anywhere `someValid(x) ? x : fallback` would be a real check. Worth adopting as a small TS idiom.
