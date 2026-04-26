---
pr: 3607
repo: QwenLM/qwen-code
title: "feat(cli): Improve custom auth wizard with step indicators and cleaner advanced config"
url: https://github.com/QwenLM/qwen-code/pull/3607
head_sha: e059fbb27906644150f8aab3b6253634a12dd75e
author: pomelo-nwu
verdict: merge-after-nits
date: 2026-04-27
---

# QwenLM/qwen-code #3607 — Custom API Key wizard with step indicators

UX rework of the Custom API Key auth wizard in the TUI: adds `Step N/6`
indicators (Protocol → Base URL → API Key → Model IDs → Advanced Config →
Review), trims redundant context per step, and redesigns the Advanced
Config screen. +1432/-16 across 8 files; the bulk is new test coverage
in `AuthDialog.test.tsx`.

## What the diff actually does

Files changed:
- `.gitignore` (+1) — adds `tmp/`. Missing trailing newline.
- `packages/cli/src/ui/AppContainer.tsx` (+3) — wires
  `handleCustomApiKeySubmit` from `useAuthCommand` through the same
  three plumbing sites used by sibling submit handlers
  (`handleCodingPlanSubmit`, `handleAlibabaStandardSubmit`,
  `handleOpenRouterSubmit`).
- `packages/cli/src/ui/auth/AuthDialog.tsx` — main UI rework, step
  state machine, step header at line 1188-1198 returning
  `'Step N/6 · <Title>'`.
- `packages/cli/src/ui/auth/AuthDialog.test.tsx` (+515) — five new test
  cases covering the wizard flow + a `typeText` helper that types
  one char at a time with 5ms delays as a workaround for "Node 24.x +
  ink compatibility issue on Windows where bulk stdin.write() may not
  propagate to TextInput correctly" (test-helper.tsx:50-62).
- `packages/cli/src/ui/auth/useAuth.test.ts` and `useAuth.ts` — exposes
  `handleCustomApiKeySubmit` through `useAuthCommand`.
- `packages/cli/src/ui/contexts/UIActionsContext.tsx` — extends
  `UIActions` with `handleCustomApiKeySubmit`.
- `packages/core/src/utils/fileUtils.test.ts` — collateral test churn
  (unclear why; flag for author).

The 6-step state machine in AuthDialog.tsx has these step IDs:
`Step 1/6 · Protocol` (OpenAI-compatible / Anthropic-compatible /
Gemini-compatible), `Step 2/6 · Base URL`, `Step 3/6 · API Key`,
`Step 4/6 · Model IDs`, `Step 5/6 · Advanced Config`, `Step 6/6 · Review`.

## Observations

1. **`.gitignore` missing trailing newline.** `.gitignore` ends at
   line 90 (`tmp/`) with no final `\n` (the diff shows
   `\ No newline at end of file`). Trivial nit, but makes the next
   `.gitignore` PR's diff noisier than necessary.

2. **`fileUtils.test.ts` change is unexplained.** The PR is scoped
   "improve custom auth wizard" but touches a `core/src/utils` test
   file. Either the test had real coupling (surprising) or it's a
   stray edit. Author should explain or split out.

3. **`typeText` workaround at AuthDialog.test.tsx:48-62 is a known
   problem with no upstream link.** Comment says "Works around a
   Node 24.x + ink compatibility issue on Windows" — please link to
   the ink/node issue tracking it, otherwise the next maintainer who
   sees this 17-line helper a year from now will reflexively try to
   delete it and break Windows CI. One-line link in the comment would
   pin it.

4. **5ms inter-character delay × N chars + 30ms suffix** can
   accumulate. For a 60-char base URL the helper sleeps ~330ms. With
   five new tests each typing multiple inputs, this adds tangible CI
   time. Worth confirming whether `await delay(1)` (forcing a tick) is
   sufficient instead of `delay(5)` — the bug is "stdin.write doesn't
   propagate" not "stdin.write needs slow-rate"; a single
   microtask boundary may be enough.

5. **Step header is built with `t('Step 1/6 · Protocol')`** at
   :1188-1196. The `·` (U+00B7) appears as `\u00B7` in source. If the
   i18n catalog translates `'Step 1/6 · Protocol'` as a single key, the
   translator can localize the entire string but can't reorder
   "Step N/M" vs the title (RTL languages may want title first). Two
   keys (`Step.label` taking the number + a separate `Step.protocol`
   taking the title) would be more flexible. Defer if the codebase
   doesn't already support compositional i18n.

6. **Test selects "Custom API Key" via "down twice from OAUTH"
   navigation** (AuthDialog.test.tsx:144-148):
   ```
   stdin.write('\u001b[B'); // Down from OAUTH -> CODING_PLAN
   stdin.write('\u001b[B'); // Down from CODING_PLAN -> API_KEY
   ```
   This wedges the test to the *current* menu order. If a future PR
   inserts a new auth type between OAUTH and CODING_PLAN, this test
   silently selects the wrong item without failing. Recommend
   targeting the menu item by visible label rather than by arrow-key
   count — `await selectMenuItem(stdin, lastFrame, 'API Key')`.

7. **`mockConfig` has only two methods stubbed** (`getAuthType`,
   `getContentGeneratorConfig`). If the real `AuthDialog` reads any
   other config method during the 6-step wizard (e.g. to validate base
   URL against an existing config) the mock will throw or return
   `undefined`. Worth confirming with a `getCurrentModel` /
   `getProxy` / etc. stub block. Tests pass today but will be brittle
   to refactors that read more config.

8. **Step 5 title "Advanced Config"** appears at lines 461 and 532 of
   the test file but the step contents aren't asserted in the diff
   chunks I read — only the header. If the redesign of step 5 is the
   advertised UX win, at least one test should assert the new layout
   (e.g. specific labels for the toggles/fields shown). Otherwise we
   only have step-header coverage.

9. **No assertion that `handleCustomApiKeySubmit` is *called*** with the
   collected values at the end of the 6-step flow. The first test
   creates `const handleCustomApiKeySubmit = vi.fn();` but doesn't
   `expect(handleCustomApiKeySubmit).toHaveBeenCalledWith(...)`. A
   single happy-path test that completes all six steps and asserts
   the submit payload is what locks the new wizard against future
   regressions — currently we're testing UI navigation, not the
   contract.

## Verdict

**merge-after-nits.** The feature is valuable (Custom API Key wizard
was a known UX papercut), the diff size is justified by the test
coverage, and the 6-step state machine reads cleanly. Asks before
merge:

1. Explain or split the `fileUtils.test.ts` change (obs 2).
2. Link the underlying ink/Node 24 stdin issue from the `typeText`
   helper (obs 3).
3. Add at least one happy-path test that completes all 6 steps and
   asserts `handleCustomApiKeySubmit` is called with the collected
   values (obs 9).
4. Replace count-based menu navigation with label-based selection in
   tests (obs 6).

Each is small.

## Banned-string scan

None.
