# sst/opencode PR #24277 — Fix team run preflight aborts (closed, scope mismatch)

- **URL:** https://github.com/sst/opencode/pull/24277
- **Author:** @terisuke
- **State:** CLOSED (auto-closed via `needs:compliance` label after 2h)
- **Base:** `dev`
- **Head SHA:** `ac3d7d7`
- **Size:** +11,374 / -55 (massive)
- **Labels:** `needs:title`, `needs:compliance`

## Summary of change

PR title says "Fix team run preflight aborts" but the actual diff
introduces an entirely new package, `packages/guardrails/`, containing
~11k lines of:

- `bin/opencode-guardrails` — a wrapper Node script that pre-sets
  `OPENCODE_CONFIG_DIR` and `OPENCODE_EXPERIMENTAL_PLAN_MODE=true`,
  then `spawnSync`s the underlying `opencode` binary
- `managed/opencode.json` — a system-managed config defaulting model
  to `zai-coding-plan/glm-5.1`, restricting `enabled_providers` to
  `["zai", "zai-coding-plan", "openai", "openrouter"]`, and disabling
  share
- `profile/` — packaged custom config dir defaults including
  `AGENTS.md`, `opencode.json`, and a guardrail plugin
- README explicitly framing this as a "Cor-Incorporated-specific layer"
  forked from upstream

The branch was opened against upstream `sst/opencode` `dev`, not
against the author's own fork.

## Findings against the diff

- **Title vs. diff mismatch (`needs:title` label is correct).** Nothing
  in the +11k lines is about "team run preflight aborts." The diff is
  a wholesale internal-distribution layer, including a bin entry that
  re-exports `opencode` under a new name. This is not a bug fix; it's
  a fork of opencode being submitted as a feature contribution
  upstream.
- **`bin/opencode-guardrails` (L1–54)** sets defaults via
  `process.env.OPENCODE_CONFIG_DIR ||= dir` (correct fall-through; user
  override wins) and then `spawnSync`s the resolved `opencode` binary.
  The resolution walks up to `packages/opencode/package.json` via
  `path.resolve(...)` — which is workspace-internal and would break
  when published as an npm package.
- **`managed/opencode.json` (L1–124)** hard-codes a single model
  (`zai-coding-plan/glm-5.1`) and a closed list of providers. As a
  managed config in upstream, this would override admin policy for
  every opencode user who happens to have this package installed —
  not appropriate for upstream. As a separately-published distribution,
  it's fine.
- **README explicitly admits this**: "should be understood as the
  Cor-Incorporated-specific layer that sits on top of upstream-compatible
  OpenCode, not as a separate reimplementation." That belongs in a
  fork, not in upstream `dev`.
- **Bot auto-closure was correct.** The `needs:compliance` label
  triggers a 2h auto-close, and the PR was closed 2026-04-25 within
  that window.

## Verdict

**request-changes**

This PR should not be reopened against upstream. The right move is:

1. Author maintains the `packages/guardrails` content in a fork
   (`Cor-Incorporated/opencode`) — which already exists per the
   README's epic links.
2. If individual upstreamable pieces exist (e.g., the
   `OPENCODE_EXPERIMENTAL_PLAN_MODE` env var support, or the wrapper-
   compatible config-dir resolution), split them into focused PRs
   with diffs of dozens of lines, not eleven thousand.
3. The bot's `needs:title` / `needs:compliance` flow handled the
   classification correctly without human reviewer cost — that's
   working as intended.

## What I learned

There's a recurring pattern in popular OSS projects where downstream
forks attempt to land their entire compliance layer upstream as a
single PR. The motive is usually "we don't want to maintain a fork."
The result is always the same: the PR is rejected, and the right
answer is the boring one — maintain the fork, upstream surgical
extension points, never upstream the policy. Bots that auto-close on
title/scope mismatch are a cheap and effective filter for this; they
spend zero reviewer attention on the (well-intentioned) wrong-shape
contribution and tell the author to come back with a tighter scope.
