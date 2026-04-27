# BerriAI/litellm PR #26573 — feat(mavvrik): add Mavvrik integration for automatic LLM spend export

- URL: https://github.com/BerriAI/litellm/pull/26573
- Head SHA: `e1afc1f73385cd066b19b5b9bcf8eeabc6545517`
- Diff: +6995 / -89 across 29 files
- Author: external contributor (pghuge-cloudwiz)
- Body marker: "🤖 Generated with Claude Code"

## Context / Problem

Adds a vendor integration with [Mavvrik](https://help.mavvrik.ai/) for
automatically exporting LiteLLM proxy spend data to a third-party cost-
management platform. Listed capabilities: zero-config startup via three
env vars, streaming page-by-page DB export (10k rows/page), gzip
on-the-fly compression, GCS chunked resumable upload, idempotent-by-
calendar-date object naming, restart catch-up via Mavvrik-owned marker,
Redis distributed lock for multi-replica deployments.

## Design — what's in the diff

10 production files under `litellm/integrations/mavvrik/`:

- `__init__.py` — public `Service` facade per endpoint
- `client.py` — Mavvrik REST (register, signed URL, advance marker,
  report error)
- `exporter.py` — DB → DataFrame → CSV via OFFSET-paginated streaming
- `uploader.py` — GCS bulk + streaming chunked upload
- `orchestrator.py` — pod-lock + catch-up date loop + advance-marker
- `logger.py` — `CustomLogger` subclass for per-request cost logging
- `settings.py` — AES-encrypted credential persistence via LiteLLM
  config store
- `_http.py` — shared httpx transport with retry + exponential backoff
- `proxy/spend_tracking/mavvrik_endpoints.py` — FastAPI router (thin
  dispatchers over `Service`)
- `types/proxy/mavvrik_endpoints.py` — Pydantic request/response models

Plus 11 test files under `tests/test_litellm/integrations/mavvrik/`
(`test_client`, `test_e2e_upload`, `test_endpoints`, `test_http`,
`test_logger`, `test_scheduler`, `test_service`, `test_settings`,
`test_transform`, `test_upload`, `test_uploader`, plus `conftest`).
Coverage shape is right — separate test per concern, including an
end-to-end upload and a transform pin.

Wiring: `litellm/__init__.py`, `litellm/constants.py`,
`litellm/litellm_core_utils/custom_logger_registry.py`,
`litellm/litellm_core_utils/litellm_logging.py`, and
`litellm/proxy/proxy_server.py` are touched — the standard
"register a new logger + add a router" pattern that other vendor
integrations follow in this repo.

Docs: `docs/my-website/docs/observability/mavvrik.md` (new) plus a
`docs/my-website/docs/proxy/config_settings.md` update for the
three env vars.

## Strengths

- **Streaming export with paged DB reads + on-the-fly gzip** is the
  right shape — claimed peak RAM ~2MB validated against 500k rows
  → 13MB / 84s (PR body). For a spend-data path that scales with
  request volume, this is the design that doesn't blow up at
  enterprise scale.
- **Idempotent-by-date GCS objects** + Mavvrik-owned marker for
  catch-up makes restart safe and double-upload-resistant. Better
  than a "last successful timestamp" stored locally.
- **Redis distributed lock** for multi-replica is correct — three
  pod replicas all running `orchestrator` would otherwise
  triple-upload. Good that it's pod-safe out of the box rather than
  documented-as-single-pod-only.
- **AES-encrypted credential persistence** via the LiteLLM config
  store (not env vars, not plaintext file) is the right secrets
  shape for a multi-tenant proxy.
- **Test coverage looks comprehensive** — 11 test files for 10
  production files, including transform and end-to-end upload. The
  project's "at least 1 test is a hard requirement" checklist box
  is unambiguously satisfied.

## Risks / nits

1. **6995 lines + 29 files for a single vendor integration** — but
   this is shaped as a self-contained subpackage under
   `litellm/integrations/mavvrik/` with mirror-tree tests, no
   cross-package surgery, and adheres to the existing
   integration-pattern in the repo (separate logger subclass,
   separate router file, separate types). Size is justified by
   completeness, not scope creep.
2. **OFFSET pagination at scale** — `exporter.py` uses
   OFFSET-paginated streaming. OFFSET on a multi-million-row spend
   table is O(n) per page (database has to skip-and-discard); a
   keyset/cursor pagination on `(created_at, id)` would scale
   better. For the claimed 500k / 84s benchmark this is fine; for
   10M+ row daily volumes it will degrade. Worth a follow-up issue
   to switch to cursor pagination.
3. **GCS-only upload destination** — the uploader is GCS-specific,
   not a pluggable storage abstraction. Other LiteLLM observability
   integrations (e.g. Arize) abstract their HTTP-transport via
   `_http.py` but here the GCS resumable-upload protocol is
   hard-coded. If Mavvrik later supports S3/Azure Blob, this will
   need refactoring. Acceptable as v1 since the PR title is
   "automatic LLM spend export" not "pluggable backend."
4. **Catch-up loop bound** — the date loop "back-fills all missed
   days from Mavvrik-owned marker." If a deployment is offline for
   30 days, that's 30 sequential days of full-table dumps on the
   first restart, blocking the orchestrator and competing with live
   query load. A configurable max-catch-up-window plus a
   `max_concurrent_dates` would protect production deployments.
5. **Retry logic** — `_http.py` documents "retry + exponential
   backoff" but the test surface in `test_http.py` should pin: max
   retry count, backoff base, backoff cap, idempotency keys (does
   each retry use the same `register` request id?). Flaky vendor
   APIs eat retried calls and double-charge or double-register
   without idempotency keys.
6. **Vendor lock-in via LLM-generated code** — the PR body
   acknowledges Claude-Code authorship. For a vendor integration
   that touches `proxy_server.py`, encrypted-credential storage,
   and per-request cost logging, the maintainer should do a
   line-by-line eyeball of `settings.py` and the `proxy_server.py`
   touch in particular, regardless of test coverage. Generated code
   for security-adjacent paths benefits from human re-derivation.
7. **No CI link in the PR description** — the project's PR
   template asks for "test plan" with `make test-unit passes`
   ticked. Confirming that on the actual CI run (not just the
   author's local) before merge.

## Verdict

`needs-discussion` — the design is sound on paper (streaming +
idempotent + pod-safe + encrypted-creds) and the test surface is
comprehensive, but a 7000-line LLM-authored vendor integration
touching `proxy_server.py`, encrypted credential persistence, and
per-request cost logging deserves an explicit maintainer
conversation before review proceeds: (a) is this vendor on the
project's accepted-integration list, (b) who owns line-by-line
review of `settings.py` (AES-encryption surface) and the
`proxy_server.py` touch, (c) is OFFSET pagination acceptable as
v1 with a documented follow-up to switch to cursor pagination,
and (d) what's the catch-up-loop max-window default — current
"all missed days" can DoS a deployment after a long outage.
Generated code for security-adjacent paths benefits from human
re-derivation regardless of test coverage.

## What I learned

A "vendor integration" PR shape that survives review: subpackage
under `integrations/<vendor>/`, mirror-tree tests, no cross-package
surgery, clear three-file public surface (`Service` facade,
`CustomLogger` subclass, FastAPI router), and pod-safe-by-default
via Redis lock rather than "single-pod only" docs. The catch:
OFFSET pagination is the default first reach and the worst choice
for spend-table-sized data — call it out in review every time.
