---
pr-url: https://github.com/block/goose/pull/8889
sha: de0678957f08
verdict: merge-as-is
---

# chore: disable spellcheck in model search

Surgical UX fix for the goose2 model picker: model IDs like `gpt-4o-mini-2024-07-18`, `claude-sonnet-4`, `o1-preview` are not English words, and the browser's default red-squiggle spellcheck on the search input was both visually noisy and (more importantly) caused some browsers' autocorrect to mangle pasted IDs. The fix at `ui/goose2/src/features/chat/ui/AgentModelPicker.tsx:245-254` swaps the bespoke 11-line `<input>` block (with its own `IconSearch` absolute-positioned overlay) for the shared `<SearchBar>` component, which already sets `spellcheck="false"`.

The 28-line test addition at `__tests__/AgentModelPicker.test.tsx:130-158` is the right shape: render the picker, click "choose agent and model", click "Browse all models", grab the search input by placeholder text, assert `toHaveAttribute("spellcheck", "false")`. That pins the actual user-facing contract (the attribute on the rendered DOM) rather than the implementation detail (which component is used), so a future refactor that swaps `<SearchBar>` back out cannot silently regress.

The diff also drops a now-redundant `IconSearch` import — the `SearchBar` component owns its own search icon. Net `+37/-11` for one component swap + one regression test is the right ratio.

Merge-as-is: zero behavioural risk for users who don't care, real visual-noise improvement for everyone else, regression-pinned at the contract boundary, follows the existing `<SearchBar>`-as-shared-component pattern that the rest of `ui/goose2/src/shared/ui/` already standardizes on.

## what I learned
The right shape for `chore:` UX nits is to pin the contract at the rendered-DOM level (`toHaveAttribute("spellcheck", "false")`) not the component level (`toBe(<SearchBar>)`) — that way the test survives the next round of component consolidation without needing to be rewritten, and it actually catches the regression a user would experience.
