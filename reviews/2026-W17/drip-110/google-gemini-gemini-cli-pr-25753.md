# google-gemini/gemini-cli PR #25753 — default e2e integration tests to Flash Preview, skip flaky interactive on macOS

- **PR**: https://github.com/google-gemini/gemini-cli/pull/25753
- **Author**: @SandyTao520
- **Head SHA**: `c14d699e833819cf3be243b8d5e318b76962dcbf`
- **Size**: +27 / −6
- **Files**: `integration-tests/context-compress-interactive.test.ts`, `integration-tests/file-system.test.ts`, `integration-tests/plan-mode.test.ts`, `packages/test-utils/src/test-rig.ts`

## Summary

Three concurrent CI-stability changes bundled as one PR:

1. `test-rig.ts` swaps the integration default model from `PREVIEW_GEMINI_MODEL` to `PREVIEW_GEMINI_FLASH_MODEL` (cheaper / faster e2e default).
2. `context-compress-interactive.test.ts` skips its entire `describe` block on `process.platform === 'darwin'`, with a long comment citing four specific run IDs of CI flake.
3. `file-system.test.ts` loosens one assertion from `expect(content).toBe('hello')` to `expect(content.trim()).toBe('hello')` — model-tolerant whitespace handling.
4. `plan-mode.test.ts` rewrites two test prompts to be more model-deterministic (explicit "Treat this as a Directive and write the file immediately without proposing strategy or asking for confirmation").

## Verdict: `needs-discussion`

Two of the four changes are healthy (Flash default; tolerant whitespace assertion). Two are *symptom suppression* that needs a conversation:

- **The `describe.skipIf(skipOnDarwin)` for the entire interactive describe.** The comment is unusually candid — it correctly identifies that the captured pty buffer contains startup escape sequences and that the failure is independent of the change under test. That's useful diagnostics, but skipping the whole block on macOS removes the only signal the project has that interactive mode works on macOS. The right cure is to fix the pty buffer capture (filter the startup banner before `expectText` runs); the pragmatic-but-wrong cure is what this PR ships. At minimum, file a tracking issue and reference it in the skip comment, and add a `RUN_FLAKY_DARWIN_INTERACTIVE=1` env override so the tests can still be run locally.
- **Rewriting prompts to be more imperative ("Treat this as a Directive ...").** This trains the test on a single model's instruction-following idiosyncrasies. When the next default model rotates in (or when Flash Preview is updated), these prompts may stop working in a way that's hard to diagnose because the test name says "Plan Mode" but the actual assertion is on a particular model's compliance pattern. Adding `args: PROMPTS.PLAN_MODE_DETERMINISTIC_FILE_CREATE` (a named constant in one place) would at least make this dependency reviewable.

The Flash-default change deserves its own PR, separately reviewable for cost / latency / regression-coverage tradeoffs.

## Specific references

- `packages/test-utils/src/test-rig.ts:14-17` — import flipped from `PREVIEW_GEMINI_MODEL` to `PREVIEW_GEMINI_FLASH_MODEL`.
- `packages/test-utils/src/test-rig.ts:478-482` — used in the integration test runtime-config block. Net: every integration test that doesn't override `model.name` now runs against Flash. CI cost ↓, but model-dependent assertions across the integration suite need a one-time validation pass.
- `integration-tests/context-compress-interactive.test.ts:11-21` — the long skip comment is excellent forensics: it names four CI run IDs and notes the failure is "across unrelated runs on `main`" and "consecutive merge-queue gates for #25753". Take the comment, file an issue, link it.
- `integration-tests/file-system.test.ts:137-139` — `expect(newFileContent.trim()).toBe('hello')`. Correct loosening — the test is named about path-with-spaces handling, not byte-perfect content.
- `integration-tests/plan-mode.test.ts:84-88, 200-204` — prompt rewrites. See discussion point above.

## Nits

1. Split into three PRs: (a) Flash default, (b) Darwin skip + tracking issue, (c) plan-mode prompt determinism + named constant.
2. The Darwin skip comment cites run IDs that will eventually 404; copy the relevant log excerpts into a tracking issue body so the diagnosis survives.
3. Consider a `// flake-tracker:` magic-comment convention so a CI job can grep for these and report "N tests are skipped due to known flakes, oldest from Y date" — gives the team a flake-debt dashboard for free.

## What I learned

This PR is a useful artifact of the slow-burn cost of pty-based interactive testing: the test infrastructure captures a buffer that includes startup-time escape sequences, and the assertion is that some output text appears in that buffer. Any startup-banner change anywhere in the CLI silently invalidates the assertion's prefix-handling. The clean fix (a `BannerFilteredPty` wrapper that drops everything before the first user-visible prompt) is the kind of infra investment that pays for itself the first time it's used; the skip-on-darwin patch is a 6-month tech-debt loan against that fix. Worth pricing the loan.
