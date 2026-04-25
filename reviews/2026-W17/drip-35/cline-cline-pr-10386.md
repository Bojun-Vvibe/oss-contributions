# cline/cline #10386 — fix: preserve multiline thinking blocks

- **Repo**: cline/cline
- **PR**: [#10386](https://github.com/cline/cline/pull/10386)
- **Head SHA**: `7d723c7ab0adcc4597418be723838a060e57311d`
- **Author**: toby-bridges
- **State**: OPEN (+12 / -1)
- **Verdict**: `merge-as-is`

## Context

Issue #7876: when an extended-thinking model emits multi-line
reasoning content, the expanded `ThinkingRow` collapses it into a
single line with no wrapping. Cause: the row's expanded body is
implemented as a `<Button>` (so the whole region remains
keyboard-focusable for collapse), and the shared button class set
includes `whitespace-nowrap` from the design-system base. The inner
`<span>` that holds the reasoning text has `whitespace-pre-wrap`
correctly, but the parent button's `nowrap` wins for layout because
the inner span's content can't wrap inside a parent that explicitly
forbids wrapping.

## Design

One-line fix at
`webview-ui/src/components/chat/ThinkingRow.tsx:92`:

```diff
- "flex gap-0 overflow-hidden w-full min-w-0 max-h-0 opacity-0 …"
+ "flex gap-0 overflow-hidden whitespace-normal w-full min-w-0 max-h-0 opacity-0 …"
```

Adding `whitespace-normal` to the expanded button explicitly overrides
the design-system `whitespace-nowrap` default, restoring normal text
wrapping. The inner `whitespace-pre-wrap` on the reasoning span then
takes over within the now-wrappable container, preserving the original
newlines as visible line breaks. This is the correct Tailwind cascade —
later utilities for the same property win at the same specificity, so
ordering inside the `cn()` call doesn't matter.

This change applies *only* to the expanded variant (the `isExpanded`
branch starting at line 90), so the collapsed/single-line state still
gets its `nowrap` behavior from the base button class. Surgical and
correct.

## Test

The new regression test at
`webview-ui/src/components/chat/ThinkingRow.test.tsx:24-33` is
exactly the right shape:

```ts
expect(reasoningContent.textContent).toBe("Step 1\nStep 2")
expect(reasoningContent.closest("div")).toHaveClass("whitespace-pre-wrap")
expect(reasoningContent.closest("button")).toHaveClass("whitespace-normal")
expect(reasoningContent.closest("button")).not.toHaveClass("whitespace-nowrap")
```

It locks down three things:

1. The text content still contains the literal newline (no DOM
   collapse).
2. The wrapping `<div>` retains `whitespace-pre-wrap`.
3. The enclosing `<button>` has `whitespace-normal` and explicitly
   does not have `whitespace-nowrap`.

That third assertion is the important one — it means if a future
refactor reintroduces `whitespace-nowrap` on the button (easy mistake
when extending the design system's button base), the test catches it
immediately. Good defensive style.

## Risks / Nits

1. **Existing biome warning preserved.** The PR author calls out the
   pre-existing `useExhaustiveDependencies` informational note on
   `ThinkingRow.tsx`. They did the right thing leaving it alone in a
   bug-fix PR; mixing a lint cleanup in here would muddy the diff.

2. **No screenshot in description.** Acceptable per the author's note
   — the linked issue already has the broken-rendering visual, and a
   single class addition with a regression test doesn't need one.

3. **Manual verification only on the affected component**, no full
   `npm run test`. For a 1-line CSS-class fix with a targeted test,
   this is fine; CI will catch any cross-component regression. If CI
   passes, no action needed here.

4. Nit: the test selector `screen.getByText(/Step 1/, { selector:
   "span" })` is a regex-loose match. Tightening to `getByText("Step
   1\nStep 2")` would be more direct but might fail because of how
   `react-testing-library` normalizes whitespace in `getByText` —
   the regex form is the safer choice. Leave as is.

## Verdict rationale

`merge-as-is`. One-line CSS fix, targeted regression test, scoped
exclusively to the expanded state, no behavior change anywhere else.
The kind of PR that should land in <24h.

## What I learned

`whitespace-nowrap` on a parent will always win against
`whitespace-pre-wrap` on a child for the purpose of *whether* lines
break — because the parent reserves no wrap point for the child to
break at. The fix is to neutralize the parent's `nowrap` rather than
escalate the child's wrap mode. Useful Tailwind / CSS rule of thumb
for any "I added pre-wrap and it still doesn't wrap" debugging.
