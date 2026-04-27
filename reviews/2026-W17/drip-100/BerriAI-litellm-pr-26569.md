# BerriAI/litellm#26569 — Litellm oss staging 04 21 2026 2

- **Author**: Sameerlite
- **Size**: +4112 / -126 (mega-PR, multi-feature staging roll)

## Summary

This is a re-attempt of #26216 (closed due to "too many conflicts") — a staging-rollup PR bundling at least the following discrete features visible in the diff:

1. **New provider: Reducto** — `docs/my-website/docs/providers/reducto.md` (+103 lines), exposed through LiteLLM's OCR surface (`reducto/parse-v3`, `reducto/parse-legacy`).
2. **New guardrail: Rubrik** — `docs/my-website/docs/proxy/guardrails/rubrik.md` (+188+ lines), tool blocking + batch logging integration.
3. Likely: provider implementation modules, guardrail hooks, OCR transform code, model registry entries.

The PR body is a stock template with **all checkboxes empty**, including:
- "I have Added testing" — **hard requirement** per the project's contributing rules.
- "Branch creation CI run" — link blank.

## Diff inspection (limited to the first 250 lines visible)

The visible portion is entirely documentation. The implementation must live further in the diff but was not reachable in the standard review window. From the docs alone:

**Reducto provider docs** are well-structured:
- Provider table with route, supported ops, models.
- SDK quick-start with `litellm.ocr()` / `litellm.aocr()` examples.
- Proxy `model_list` config example with `mode: ocr`.
- Per-model section for `parse-v3` (`formatting`, `retrieval`, `settings` params) and `parse-legacy` (`enhance` param).
- Upload-behavior section explicitly documents three `document=...` shapes: file (auto data-URI + `/upload` then `/parse`), `reducto://` URLs (passthrough), `http(s)` URLs (rejected).
- Cost tracking via `ocr_cost_per_credit` registration pattern.

**Rubrik guardrail docs** are well-structured:
- Tabs UI (`config.yaml` recommended vs env vars).
- `mode: post_call`, `default_on: true` config shape.
- Fail-open semantic explicitly documented ("If the tool blocking service is unavailable, requests are allowed through unchanged.") — important security-posture statement.
- Env var fallback chain with `RUBRIK_WEBHOOK_URL`, `RUBRIK_API_KEY`.

## Strengths

- Documentation quality on both new providers is high — config examples, env var fallback patterns, security-posture (fail-open) explicitly called out.
- Reducto's three-document-shape upload behavior matrix is the kind of edge-case documentation that prevents user issues post-launch.
- `RUBRIK_WEBHOOK_URL` env var name follows project convention.

## Concerns

1. **PR title is meaningless.** "Litellm oss staging 04 21 2026 2" tells reviewers nothing about scope. A staging-rollup PR should at minimum enumerate "adds Reducto OCR provider, Rubrik guardrail, [other features]" in the title or first line.
2. **PR body is a stock template with no scope summary.** "Reraising clean oss staging as #26216 had too many conflicts" is the only substantive line — doesn't tell reviewers what features are in scope, what was dropped from #26216, what conflicts were resolved how.
3. **All required-checklist boxes are empty.** Per the PR template's own bold-text requirement: "Adding at least 1 test is a hard requirement." Without seeing the full diff I can't confirm tests exist, but the unchecked box is a maintainer-facing red flag.
4. **Multi-feature staging-rollup PR violates the "isolated scope" checkbox** that the template itself ranks as one of four hard requirements ("My PR's scope is as isolated as possible, it only solves 1 specific problem"). At 4112 / -126 across at minimum a new OCR provider AND a new guardrail AND likely several other deltas, this is ~5+ logical PRs jammed together.
5. **Bisect granularity is destroyed.** When a future regression is traced to "something in #26569", git-bisect lands inside this single squash and the next reviewer has to read 4k lines to localize. Splitting into per-feature PRs (Reducto, Rubrik, etc.) would make every individual change ~300-700 lines and bisect-friendly.
6. **Backport granularity is destroyed.** A user wanting just the Reducto provider on an older release branch can't cherry-pick this — they get the Rubrik guardrail and everything else too.
7. **CI status link blank.** Without a passing CI badge or test-count summary in the PR body, reviewers can't gauge regression risk.
8. **Reducto OCR docs reference `ocr_cost_per_credit`** — need to confirm this hook actually exists in the cost-tracker before docs ship promising it.
9. **Rubrik fail-open semantic is a security choice that deserves a CODEOWNERS-level review.** Most guardrail integrations should fail-CLOSED for safety; fail-open is appropriate for *availability-critical* deployments but the choice should be configurable, not hardcoded.

## Verdict

**request-changes** — split this into discrete per-feature PRs:
1. `feat(reducto): add OCR provider` — provider impl + docs + tests
2. `feat(rubrik): add tool-blocking + batch-logging guardrail` — guardrail impl + docs + tests + fail-open-vs-fail-closed config
3. Each remaining feature in its own PR

Each split should fill in the required-test checkbox and link a passing CI run. The current PR shape (4k-line "staging rollup" with a meaningless title and empty checklist) is fundamentally unmergeable regardless of the underlying code quality, because it destroys bisect/backport granularity and fails the project's own "isolated scope" hard requirement.
