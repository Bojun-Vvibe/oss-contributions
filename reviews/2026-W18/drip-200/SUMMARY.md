# drip-200 — 2026-04-30

8 fresh PRs across 5 tracked upstream repos (2 sst/opencode, 2 openai/codex, 2 BerriAI/litellm, 1 google-gemini/gemini-cli, 1 block/goose; QwenLM/qwen-code skipped — every recent open candidate was already covered in prior drips).

**Theme-of-the-drip:** "module-boundary moves and trust-gating land cleanly; one spam-shaped PR and one architecture-RFC bookend the drip."

## Verdict mix

- **3 merge-as-is**: openai/codex #20342 (header-only context — see actual file: see drip-200/openai-codex-pr-20342.md, marked merge-after-nits — corrected mix below)

(authoritative count below)

- **1 merge-as-is**: sst/opencode #25047 (eighth-entry testEffect migration matching the established `it.live` + `Effect.gen` shape, byte-identical assertion semantics, with hardening on the previous implicit non-null assertions via `NamedError.Unknown` fallthrough)
- **5 merge-after-nits**:
  - sst/opencode #25081 (scrollable-share-list rewrite is correct but `no-scrollbar` on the long surface needs a discoverability affordance, per-turn-model-header drift needs explicit confirmation, and a CHANGELOG entry is missing for the lost per-message deep-link)
  - openai/codex #20342 (config-layer feature/tool-suggest helpers extraction is the right move and the `Features::from_sources` carve-out doc-comment is load-bearing, but the `disabled_tools` ordering invariant differs between the empty-layers and populated-layers paths and needs a regression that pins both, and the `toml`/`toml_edit` co-presence in `codex-config` deps wants a one-line rationale)
  - openai/codex #20339 (linux-sandbox protected-metadata-runtime extraction is byte-identical at the new location for the visible deletions, but `BWRAP_CHILD_PID`/`PENDING_FORWARDED_SIGNAL` becoming module-level statics deserves a single-shot doc, the `FORWARDED_SIGNALS = [SIGHUP, SIGINT, SIGQUIT, SIGTERM]` exclusion of `SIGUSR1`/`SIGUSR2`/`SIGWINCH`/`SIGPIPE` needs a comment naming the rationale, and the git-worktree `.git`-as-file fallback in `should_leave_missing_git_for_parent_repo_discovery` should be re-verified at the new location)
  - BerriAI/litellm #26870 (provider-cost streaming propagation is the right shape with both attribute and dict access, the `additional_headers` key matches the non-streaming convention, and the `test_openrouter_streaming_cost_propagated_to_final_response` regression locks both `Usage.cost` and the header-shape, but the same 6-line block lives at `main.py:7462` AND `:7646` and should be a `_propagate_streaming_response_cost` helper, plus a negative test for the absent-cost case is missing)
  - BerriAI/litellm #26861 (SCIM-deprovisioning kill-switch closes a real cross-trust gap with the `is False` identity-comparison gate that correctly distinguishes "explicitly deactivated" from "never set," and `_set_user_keys_blocked` does the right prisma + cache-invalidate pair, but the cache-invalidate path needs the `task.add_done_callback(_log_exc)` pattern from drip-199 #26859 to surface redis outages, and the unrelated `_experimental/out/*.html` → `*/index.html` route-shape change should be split into its own PR)
- **1 request-changes**: google-gemini/gemini-cli #26251 (the diff prepends `import clang`/`import dotnet`/`import pwsh`/literal-backslashes to a TOML file — these are not valid TOML 1.0.0 grammar and the file will fail to parse on first non-comment line, breaking the `/review-frontend` slash command for every user; PR has no description, no test, no linked issue, and pattern-matches accidental/spam class — recommend close with note plus a CI lint that runs `tomli` parse over `.gemini/commands/*.toml` on every PR)
- **1 needs-discussion**: block/goose #8928 (well-structured `+569`-line ACP-session-id architecture-review markdown correctly identifies the canonical invariant `ChatSession.id == ACP sessionId == Goose Thread.id` and the failure-driven control flow in `acpCreateSession()` plus the `acpSessionId`/`id` duplication, but a `+2483` docs-only PR with no companion tracking issue, no enumeration of which other markdown files are included, no named owner, and no end-of-life condition risks becoming architectural-debt-as-documentation — needs path relocation under `docs/rfcs/...`, a tracking issue link, and an explicit "delete when X" condition before merge)

**Authoritative mix: 1 merge-as-is / 5 merge-after-nits / 1 request-changes / 1 needs-discussion.**

## Repo coverage

5 distinct upstream repos. Three of the eight close real cross-trust or money-on-the-table gaps (litellm #26861 SCIM-deprovisioned virtual-key kill-switch, litellm #26870 OpenRouter streaming cost field silently dropped during reassembly, codex #20339 linux-sandbox structural extraction). One is the architecture-RFC ahead of a goose2-session-id cleanup series (goose #8928). One is a spam-shaped TOML-breaking PR worth flagging publicly (gemini-cli #26251). The opencode pair tail-ends both the share-page UX cleanup and the testEffect migration series.
