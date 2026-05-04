# google-gemini/gemini-cli PR #26465

- **Title**: feat(cli): support @-mentioning files in AskUser custom input
- **Author**: Adib234
- **Head SHA**: `327ba49b3d80c068e35bddcd4c91bc7acf1f4bf8`
- **Diff**: +103 / -3

## Summary

Adds `@`-mention file autocomplete to the AskUser dialog by introducing a new `AutocompleteTextInput` wrapper that composes `TextInput` + `useCommandCompletion` + `SuggestionsDisplay`, and swaps both `TextQuestionView` and `ChoiceQuestionView` (custom input branch) to use it.

## File-by-file

### `packages/cli/src/ui/components/AskUserDialog.tsx` (L9, L18-25, L31-39)

Two call-site swaps from `TextInput` → `AutocompleteTextInput`. Both pass `availableWidth` and `suggestionsPosition="below"`. Reasonable choice for a modal — suggestions above would overlap the question text.

### `packages/cli/src/ui/components/shared/AutocompleteTextInput.tsx` (new, L47-143)

- **L84-93 `useCommandCompletion`**: passes `slashCommands: []` and `commandContext: {} as unknown as CommandContext`. The cast suppression at L88-89 is honest about the hack. This works because `@`-mention completion path doesn't read `commandContext`, but it is a load-bearing assumption — if `useCommandCompletion` ever starts touching `commandContext` for any code path that's reachable here (e.g. mixed `/`+`@` parsing), this silently NPEs in production. Worth either: (a) asserting at runtime, (b) gating completion to file-only mode via a new prop on the hook.
- **L95-115 `handleKeypress`**: handles Tab (autocomplete), Up/Down (navigate), and Enter when not a perfect match (autocomplete instead of submit). Returns `false` when no suggestions visible — correct for chained handlers. Note Enter on a perfect match still falls through to `TextInput.onSubmit`, which is the right UX.
- **L117-120**: `priority: true` on `useKeypress` is correct — autocomplete needs to intercept Tab/Enter before the default `TextInput` handler runs.
- **L122-134 `SuggestionsDisplay`**: wired with `mode="reverse"` and `width={availableWidth}`. Default `availableWidth = 80` at L78 is a sane fallback but will look wrong on narrow terminals if a caller forgets to thread the real value.

## Risks

- Empty `slashCommands` and stub `commandContext` means anything in `useCommandCompletion` that gracefully degrades today might start behaving differently. Recommend a smoke test that opens the AskUser dialog, types `/` and confirms it doesn't crash or show stale slash suggestions.
- No new tests in this diff for `AutocompleteTextInput` itself. A snapshot/render test for "@-typed → suggestions render below input" would lock the contract.

## Verdict

**merge-after-nits**
