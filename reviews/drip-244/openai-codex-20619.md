# PR #20619 — [codex] request desktop attestation from app

- Repo: openai/codex
- Head: `cdff8cc17678e84c3f6089a5ddea4fb7301197da`
- URL: https://github.com/openai/codex/pull/20619
- State: DRAFT
- Verdict: **needs-discussion**

## What lands

+1020 / −14 across the app-server protocol, schema, and core. Three additions:

1. New v2 `attestation/generate` server request — adds
   `AttestationGenerateParams` (empty object, just a request marker) and
   `AttestationGenerateResponse` as a tagged union of `TokenAttestationGenerateResponse`
   `{ type: "token", token: string, latencyMs?: number }` and
   `FailureAttestationGenerateResponse` `{ type: "failure", failureReason:
   string, failureDetail?: string, latencyMs?: number }`. Generated schemas at
   `app-server-protocol/schema/json/{AttestationGenerateParams,AttestationGenerateResponse}.json`
   and registered in `ServerRequest.json:124-126,1903-1928`.

2. Codex core asks the connected desktop app for a fresh attestation
   immediately before protected ChatGPT Codex requests.

3. Forwards the existing `x-oai-attestation` payload shape for responses,
   compaction, and realtime setup.

Companion desktop-app PR linked in the description (`openai/openai#878649`).

## Why needs-discussion (not a code-quality verdict)

This is **DRAFT** and the author has marked it as such — Codex asking the
desktop app for a fresh attestation before *every* protected request is a
non-trivial coupling that deserves discussion before review effort goes deep:

- **Latency budget**: every protected request now blocks on a round-trip to
  the desktop app. The `latencyMs` field on both response shapes suggests the
  author is aware, but the PR body has no SLO discussion or fallback policy.
  What happens when the desktop app is slow, frozen, or doesn't respond inside
  N ms — does codex hard-fail the user's request, fall back to a cached token,
  or proceed without attestation?
- **Failure semantics**: `FailureAttestationGenerateResponse` carries
  `failureReason: string` (free-form) — this needs to be an enum (or at least
  a documented set of canonical values) so callers can handle
  `user_declined` vs `tpm_unavailable` vs `transient_error` differently.
  String matching is a forever bug source.
- **Headless / non-desktop hosts**: codex is also used outside the desktop
  app (CI, server contexts). The PR title says "from app", but the protocol
  change lands in `app-server-protocol` which is consumed by both flows.
  Behavior when there's no desktop attached deserves explicit doc text in the
  schema.
- **Schema-only PR shape**: 21 of the 28 changed files are generated schemas
  (`schema/json/*.json`, `codex_app_server_protocol.schemas.json`,
  `ServerRequest.json`). The hand-written core/app-server logic is the
  load-bearing review surface — would help reviewers if the PR body explicitly
  called out which 7 files are hand-written vs which are generated drift.

## Recommendation

Move out of DRAFT only after answering: (1) timeout / fallback policy,
(2) `failureReason` enum or canonical-value list, (3) headless-host behavior,
(4) PR-body callout splitting hand-written vs generated diff.