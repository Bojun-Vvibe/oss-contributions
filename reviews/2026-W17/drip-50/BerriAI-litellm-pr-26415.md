---
pr: 26415
repo: BerriAI/litellm
sha: 7a24f4ec6ce02f8082ca35b58d119ed483390b4e
verdict: needs-discussion
date: 2026-04-25
---

# BerriAI/litellm#26415 — feat(mavvrik): add Mavvrik integration

- **URL**: https://github.com/BerriAI/litellm/pull/26415
- **Author**: pghuge-cloudwiz

## Summary

Adds a complete LiteLLM-proxy → Mavvrik (third-party AI cost
management SaaS) integration: streamed CSV export of
`LiteLLM_DailyUserSpend` page-by-page, gzip-compressed and uploaded
to GCS via 256KB chunked resumable upload, scheduled hourly with a
Redis-backed distributed lock for multi-replica deployments, plus
six admin REST endpoints (`/mavvrik/init`, `/settings`,
`/dry-run`, `/export`, `/delete`). Total: +5,706 lines, 28 files,
153 unit tests + 1 e2e file. Three required env vars
(`MAVVRIK_API_KEY`, `MAVVRIK_API_ENDPOINT`,
`MAVVRIK_CONNECTION_ID`).

## Reviewable points

- The PR is structurally clean: a self-contained
  `litellm/integrations/mavvrik/` package (`__init__.py`,
  `_http.py`, `client.py`, `exporter.py`, `logger.py`,
  `orchestrator.py`, `settings.py`, `uploader.py`) plus a
  proxy-side `litellm/proxy/spend_tracking/mavvrik_endpoints.py`
  and a `litellm/types/proxy/mavvrik_endpoints.py` type module.
  Exactly one touchpoint each in `litellm/__init__.py:+1`,
  `litellm/constants.py:+6`, `litellm/proxy/proxy_server.py:+45`,
  `litellm/litellm_core_utils/litellm_logging.py:+15`, and
  `litellm/litellm_core_utils/custom_logger_registry.py:+2`. That's
  the right footprint shape for an opt-in observability backend.

- `litellm/proxy/proxy_server.py:+45` is the bit to scrutinize.
  Adding 45 lines to `proxy_server.py` for a single third-party
  integration is on the high side for this project — the historical
  pattern (other observability integrations: arize, langfuse,
  newrelic, etc.) is closer to 5–10 lines plus a hook in
  `litellm_logging.py`. Worth checking that those 45 lines are
  scheduler-registration glue (which would be one-time at startup)
  and not per-request hot-path code.

- `litellm/litellm_core_utils/litellm_logging.py:+15` is the
  per-request injection point. 15 lines here is normal for a new
  custom logger, but specifically verify whether any of those
  lines run on the synchronous request path vs. only on the async
  background flush — Mavvrik's documented value-prop is daily
  aggregate export, so there should be *zero* synchronous overhead
  on the request hot path. If `_set_usage_outputs`-style
  per-request hooks were added, that's a regression vs. the design
  goal.

- `litellm/integrations/mavvrik/uploader.py:+232` and
  `client.py:+188` implement the GCS chunked resumable upload and
  the Mavvrik HTTP client. With ~1700 lines of test coverage
  (`test_uploader.py:+562`, `test_upload.py:+393`,
  `test_endpoints.py:+457`, `test_settings.py:+487`,
  `test_scheduler.py:+400`, `test_transform.py:+477`,
  `test_service.py:+299`, `test_http.py:+215`, `test_logger.py:+24`)
  the unit-test side is unusually well-covered for a litellm
  community contrib.

- `litellm/integrations/mavvrik/orchestrator.py:+189` plus the
  Redis distributed lock claim — verify what happens when the lock
  acquisition itself fails (Redis unreachable, replica with a
  partial lease). The natural failure mode is "skip this run, try
  next hour", but if the orchestrator instead crashes the
  scheduler thread the integration silently dies on the first
  Redis hiccup. `test_scheduler.py:+400` should cover this; worth
  a quick scan.

- The admin endpoint surface (6 endpoints in
  `mavvrik_endpoints.py:+224`) is gated to "proxy admin" — confirm
  the `init` and `settings` endpoints encrypt credentials at rest
  the same way the proxy already encrypts other third-party
  credentials (e.g. via `litellm.proxy.utils.encrypt_value` or
  whatever the project's standard is). The PR description says
  "stores encrypted credentials in the database" but doesn't say
  through which encryption helper.

- Naming: the integration uses `Mavvrik` consistently (no shadow
  spelling like "Maverick"), the env-var prefix is `MAVVRIK_`, the
  endpoint root is `/mavvrik/`. Internally consistent. No collision
  with existing integrations that I can see.

## Rationale

The code structure, test density, and documentation
(`docs/my-website/docs/observability/mavvrik.md:+237`) are all
genuinely good for a third-party-vendor contribution. The
needs-discussion verdict is about scope, not quality:

1. This is a +5,706-line PR adding a third-party SaaS integration
   to a project where similar integrations have historically been
   ~500–1500 lines. The maintainers should decide whether
   `litellm/integrations/<vendor>/` is a sustainable pattern for
   8-file vendor SDKs vs. encouraging vendor-side webhook
   consumption of the existing logging callback shape.

2. The Redis distributed-lock + scheduler + chunked-upload stack
   is real production infrastructure; if it breaks (e.g. GCS
   resumable-upload session expiry, marker mismatch with Mavvrik
   API), the failure mode could be either "silent: no exports for
   N days" or "loud: every hourly run pages an on-call". Confirm
   the alerting story before merge.

3. The 45-line `proxy_server.py` delta needs justification (or
   consolidation behind a helper in `litellm_core_utils`).

If the maintainers are happy with the scope on those three axes,
this is a merge-after-nits. As-is I'd want answers to those before
the green button.

## What I learned

There's a recurring tension in agent-infra projects between "every
vendor integration is a new package under integrations/" (good for
discoverability, bad for surface-area / maintenance debt) vs.
"vendors should consume our generic webhook/callback shape" (good
for the project, bad for vendor adoption since their customers want
turnkey). LiteLLM has historically chosen path #1 and this PR
extends that pattern further than usual. The correctness lessons
embedded here are also worth noting: (a) for SaaS-cursor exports,
making the cursor *server-owned* (Mavvrik holds `metricsMarker`,
LiteLLM is stateless w.r.t. it) eliminates a whole class of
"client thinks it's caught up but server reset" bugs at the cost of
an extra round-trip per scheduler run; (b) idempotent-by-date
object naming (`gs://.../2024-01-15`) makes re-export safe, which
is the right primitive for any "export aggregates" integration.
