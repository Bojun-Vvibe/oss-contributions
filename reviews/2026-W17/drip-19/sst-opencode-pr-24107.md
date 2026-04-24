---
pr_number: 24107
repo: sst/opencode
head_sha: 0a0b451689943740db3493a230b25d0a89e38ce7
verdict: merge-as-is
date: 2026-04-25
---

# sst/opencode#24107 — clamp question custom-input textarea height

**What changed.** 2 files / +3 / −2. `packages/app/src/pages/session/composer/session-question-dock.tsx` line 376: `el.style.height = ${el.scrollHeight}px` → `el.style.height = ${Math.min(el.scrollHeight, 120)}px`. `packages/ui/src/components/message-part.css` line 1093: `overflow: hidden` → `overflow-y: auto; max-height: 120px`. Closes #24109.

**Why it matters.** In the web/serve mode, when a model's `question` tool prompts the user and they pick "Type your own answer", a long pasted answer was pushing the Submit/Next/Dismiss footer buttons off-viewport because the textarea grew unbounded. Pure layout regression in user-facing flow.

**Concerns.**
1. **120px is a magic number duplicated in JS and CSS.** The `Math.min(el.scrollHeight, 120)` in TSX must stay in sync with `max-height: 120px` in CSS. Extract to a shared CSS custom property (`--question-input-max-height: 120px`) and read it in JS via `getComputedStyle(el).getPropertyValue(...)` or pass as a prop. As-is, a future bump touches one and silently desyncs the other.
2. **`resizeInput` is called on input/paste events** (existing behavior). After the textarea reaches 120px and overflows internally, subsequent calls still write `el.style.height = '0px'` then re-measure `el.scrollHeight`, which causes a single-frame flash of the input collapsing before re-expanding to 120px. Visible on slow paste of a long block. Tolerable but worth noting.
3. **No regression test** — the dock has tests in this repo for the option-selection flow but not for the custom-input textarea sizing. A snapshot test on computed `el.style.height` after a synthetic paste of a 50-line string would prevent the next `overflow: hidden` regression.
4. **120px ≈ 6 lines at default font-size.** That's reasonable but on small viewports (mobile-web in serve mode) it may still consume too much vertical space. Consider `min(120px, 30vh)` if mobile-web is in scope.

Tiny, correct, ships a real bug that shipped to users. Land as-is and file a follow-up for the magic-number consolidation.
