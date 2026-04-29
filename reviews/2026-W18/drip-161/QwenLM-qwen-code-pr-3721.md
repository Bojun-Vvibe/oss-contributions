# QwenLM/qwen-code PR #3721 — fix(cli): bound SubAgent display by visual height to prevent flicker

- Repo: `QwenLM/qwen-code`
- PR: https://github.com/QwenLM/qwen-code/pull/3721
- Head SHA: `6eabab1907fa40c2feb3879fa90a7d6ae4c460bb`
- State: OPEN, +1119/-50 across 8 files

## What it does

Migrates `AgentExecutionDisplay` (the SubAgent runtime panel) onto the same visual-height / visual-width slicing primitives that `ToolMessage` and `ConversationMessages` already use after PRs #3591 and #3713. Before: hard-coded `MAX_TASK_PROMPT_LINES = 5` / `MAX_TOOL_CALLS = 5` plus character-length truncation (`firstLine.length > 80`) — on narrow terminals the soft-wrapped content overflowed the available height as the tool-call list grew, forcing Ink to clear-and-redraw on every update (visible flicker on tool addition / mode switch / resize).

The fix replaces the line-count + char-length heuristics with `sliceTextByVisualHeight` so the panel honors actual rendered height in cells, regardless of wrap. This means in-place updates instead of clear-screen redraws, which is the actual flicker root cause.

## Specific reads

- `integration-tests/terminal-capture/subagent-flicker-regression.ts` (new, +617 LOC) — a deterministic TUI ratchet that's the most impressive part of this PR. The header docstring (lines 1-58) is a model for how to write a regression-ratchet test:
  - Spec'd inputs: 60×18 PTY, mock OpenAI server returning agent → 5×`read_file` → final, then main-loop final.
  - Spec'd measurement: counts `\x1b[2J\x1b[3J\x1b[H` clear-terminal triples, erase-line, cursor-up.
  - Honest documentation of what the fix *does* and *doesn't* do: "clear-pair count is *higher* with the fix because the new 'Showing N visual lines' footer + bounded slicing trigger extra commits in this scenario. That is *not* regression — the fix's primary value is preventing the flicker storm that fires when the terminal width changes mid-run, which this scenario doesn't exercise (TerminalCapture in this branch lacks a `resize()` public method)." That kind of self-aware caveat in a perf test is rare and correct.
  - Reference numbers given for both with-fix and without-fix at `M2 Pro Mac, 60-col / 18-row, 5 tool calls`: `clearTerminalPair=5/2`, `clearScreen=10/4`, `eraseLine=440/469`, `cursorUp=132/134`. The eraseLine drop is the in-place-update efficiency win.
  - Thresholds set at ~2× the with-fix steady state (`QWEN_TUI_E2E_MAX_CLEAR_PAIRS=10`, `QWEN_TUI_E2E_MAX_CLEAR_SCREEN=20`) — the ratchet trips on regression spikes (e.g., accidentally bringing back the 300ms width-driven `refreshStatic`) without being so tight that minor refactors cause false positives.
- The "without-fix sees *more* clear-pairs" caveat is load-bearing because it pre-empts the obvious reviewer question ("if this fixes flicker, why does the ratchet's clear-count go up?") — answered upfront with the resize-not-exercised explanation. Cross-reference to "the resize regression is covered by the `does not clear the terminal just because width changed` AppContainer test" closes the loop.
- The bulk of the production change (presumably ~7 files outside the test) is the swap from `MAX_TASK_PROMPT_LINES` / `MAX_TOOL_CALLS` constants to calls into the shared `sliceTextByVisualHeight` helper. Net production code is +500 LOC across multiple files — confirm the helper isn't being *re-implemented* per call site (the fix's whole point is reusing the same primitive `ToolMessage` and `ConversationMessages` use after #3591/#3713).

## Risk

- The 617-LOC integration test boots a full mock OpenAI server, drives a real PTY, and counts ANSI escapes — that's a lot of moving parts that can flake on CI under load (timing-sensitive PTY reads, Ink commit batching). The reference numbers come from "M2 Pro Mac" — a cold CI runner could trivially produce more `eraseLine` events than the threshold allows. Worth a CI-only override knob (`CI=1` → 3× thresholds) or a stability-pass requirement before counting failures.
- `npm run build && npm run bundle` is a precondition (per the docstring) — if this test runs in a pipeline that doesn't re-bundle after a code change, it silently tests the old binary. Worth a `package.json` script wrapper that bundles + runs as one unit.
- The `Showing N visual lines` footer is a new always-present UI element. Confirm it's not rendered when `N === totalLines` (the "nothing was clipped" case), otherwise users on tall terminals get a redundant noise line at the bottom of every SubAgent panel.

## Verdict

`merge-after-nits` — substantive and well-engineered fix. The integration-test docstring is exemplary (states inputs, measurements, with/without-fix numbers, and the caveat about what the test *doesn't* cover, all upfront). Three nits: (1) add a `CI=1` thresholds-relaxation knob to the regression ratchet to prevent CI flake from cold-runner timing, (2) wrap the `npm run build && npm run bundle && npx tsx subagent-flicker-regression.ts` sequence in a single `package.json` script so a CI step can't accidentally test stale binaries, (3) suppress the "Showing N visual lines" footer when nothing was actually clipped. Optional: a follow-up to add `TerminalCapture.resize()` so the next iteration of the ratchet can exercise the resize-flicker path that this fix's *primary* value addresses.
