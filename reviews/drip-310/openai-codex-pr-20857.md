# openai/codex #20857 — Add vanilla context mode

- PR: https://github.com/openai/codex/pull/20857
- Author: gustavz
- Head SHA: `fc10fd1e758171d7bfb9ed0e43853b5e970ffd4d`
- Updated: 2026-05-03T14:08:34Z

## Summary
Introduces a `ContextMode` enum (`default`, `vanilla`) plumbed through the app-server protocol on `ThreadStartParams`, `ThreadResumeParams`, and `ThreadForkParams`. In `vanilla` mode, the agent suppresses *all* custom context enrichment: configured instructions, AGENTS.md, skills, apps/plugin instructions, environment context, and personality. Effective mode is persisted in rollout session metadata so a thread that started or forked as vanilla stays vanilla on subsequent resumes when the client doesn't resend `contextMode`.

## Observations
- Schema additions (`codex-rs/app-server-protocol/schema/json/ClientRequest.json` and the v2 schemas) add `ContextMode` as a top-level enum with `default`/`vanilla` and add a nullable `contextMode` field to all three thread-lifecycle param shapes. The schema-update lands at lines 533-540 (definition) and lines 3464, 3878, 4066 (param fields) — and mirrored in `codex_app_server_protocol.schemas.json` and `codex_app_server_protocol.v2.schemas.json`. Schema changes look symmetric across v1 and v2, good.
- The nullable shape (`anyOf: [ContextMode, null]`) is the right backward-compat choice: existing clients that omit the field continue to behave as `default`, and resume/fork can distinguish "explicitly set to default" from "not specified, inherit persisted".
- The "inherit on resume/fork when omitted, preserve explicit overrides" semantics described in the PR body are the correctness story. The diff snippet shown is schema-only — the actual Rust resume/fork inheritance logic lives in `codex-app-server` / `codex-thread-store` (not in the diff window). Reviewer should specifically read those crates' patches and verify:
  1. On resume, if `contextMode` is `None`, the rollout's persisted mode is applied.
  2. On resume, if `contextMode` is `Some(default)`, that *explicit* override beats the persisted `vanilla` (i.e., the user can opt back in to enrichment).
  3. On fork, the same precedence applies, and the fork persists its effective mode independently of the parent.
- PR lists `cargo test -p codex-app-server apply_context_mode`, `cargo test -p codex-core vanilla_context_mode`, `cargo test -p codex-rollout test_base_instructions` — good coverage map. Worth specifically having a test for "fork with no `contextMode` from a vanilla parent stays vanilla after a subsequent resume" — that's the bug class the persisted-metadata design exists to prevent.
- The design choice to suppress enrichment *including* `AGENTS.md` and personality is a strong one. Confirm the docs (or a follow-up PR) make it explicit that vanilla mode is a near-raw-model surface — operators who lean on AGENTS.md for security-critical guardrails (e.g. denylists for risky shell commands) need to know vanilla mode bypasses those.
- "Kept running-thread resume-by-path and archived live-rollout reads correct across Windows path normalization differences" — that's a separate concern bundled into this PR. From a reviewer perspective, this is the second-most-likely place for a subtle regression. A targeted Windows test (the PR body lists `codex-rollout test_base_instructions` but no Windows-specific test) would help; otherwise resume-by-path on Windows can silently regress and only surface when a Windows user complains.
- Desktop UI follow-up referenced as `openai/openai#883225` — internal repo, not visible to this review. Acceptable as long as the protocol change merges before the UI starts depending on it.
- No banned strings (no third-party product names) in the diff; "openai" appears as the org/crate name only.

## Verdict
`merge-after-nits`
