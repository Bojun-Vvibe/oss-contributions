# BerriAI/litellm #27182 — feat(integrations): add Tickerr callback for LLM failure reporting

- **Head SHA:** `8047392b2161b97ab88e4c8de7fd5d95279826a1`
- **Base:** `main`
- **Author:** imviky-ctrl
- **Size:** +667 / −0 across 5 files (`integrations/tickerr.py`, registry wiring, callback_configs.json, docs, tests)
- **Verdict:** `request-changes`

## Summary

Adds Tickerr as a built-in LiteLLM callback that POSTs LLM failure
metadata (provider, model, status code, latency) to
`https://tickerr.ai/api/v1/report` whenever a request fails. Activated
via `litellm.callbacks = ["tickerr"]`. Self-described as a "crowd-sourced
outage radar for AI agents."

## What's right (the implementation isolated)

- **Non-blocking dispatch with concurrency cap.**
  `tickerr.py:135-167` wraps the POST in a daemon thread guarded by a
  semaphore (`_inflight = threading.Semaphore(_MAX_INFLIGHT)`,
  `_MAX_INFLIGHT = 5`), so a burst of 100 errors/s cannot exhaust the
  thread pool — `_inflight.acquire(blocking=False)` returns false and
  the report is silently dropped. Correct shape for a side-channel
  reporter.

- **Semaphore release on thread-start failure** at
  `tickerr.py:163-167` — if `t.start()` raises (OS thread limit), the
  semaphore slot is released so future reports aren't permanently
  blocked. This is an easy bug to miss; the author got it right.

- **`_send` is bare `except Exception: pass`** at `tickerr.py:152-154`
  — correct for a callback that must never crash the LLM call path,
  even if the network is partitioned.

- **`urllib.request` + 5s timeout** at `tickerr.py:148-150` — no
  `requests`/`httpx` dependency added, timeout prevents indefinite
  hang.

- **Provider normalization is conservative** at `tickerr.py:64-101`:
  explicit map for known providers, regex fallback for prefix
  detection (`gpt`/`o[1-9]` → openai, `claude` → anthropic, etc.),
  default to `"unknown"` rather than guessing wrong.

- **Error-code map is intentionally narrow.** `tickerr.py:54-62` calls
  out that 500 is excluded because "500 is intentionally excluded: it
  is a generic 'Internal Server Error' that indicates a crash or bug,
  not a capacity/overload condition" — careful semantic judgment.

- **14 unit tests** per PR body, plus full docs page.

## Why this needs changes anyway

**Privacy/governance is the blocker, not implementation quality.**

1. **Default-on data exfiltration to a third-party SaaS with no opt-in
   prompt.** Setting `litellm.callbacks = ["tickerr"]` is a one-liner
   that ships every failure's `(provider, model, status_code,
   latency_ms, region, client_tier)` tuple to a single external
   endpoint owned by an unaffiliated party. For LiteLLM proxy
   deployments handling regulated workloads (HIPAA, GDPR EU-residency,
   SOC2 Type 2) this is a data-handling change that needs a security
   review, not just a code review. Compare to existing callbacks like
   `langfuse` / `lunary` / `helicone` — those *all* require an explicit
   API key to activate, which acts as the explicit-consent gate.
   Tickerr requires no key.

2. **Endpoint hardcode prevents enterprise inspection.**
   `_REPORT_URL = "https://tickerr.ai/api/v1/report"` at
   `tickerr.py:30` has no env-var override. Enterprise SOC teams that
   want to (a) proxy through their egress gateway for inspection or
   (b) point the callback at a self-hosted Tickerr-compatible endpoint
   for audit logging, can't. Standard pattern in the codebase is
   `os.environ.get("TICKERR_REPORT_URL", "https://tickerr.ai/...")`.

3. **No documentation of what Tickerr does with the data.** The
   `tickerr.py` docstring says "anonymous" but the payload includes
   `client_tier` (free/pro/enterprise) and `region` (e.g. `us-east-1`)
   from env vars — this is fingerprinting metadata, and combined with
   request timing patterns can de-anonymize a deployment to anyone with
   correlating telemetry. The Tickerr privacy policy / data-retention
   policy isn't linked from `tickerr.py` or
   `docs/observability/tickerr.md`. For a built-in callback this needs
   to be visible to the user *before* they enable it.

4. **No domain allowlist / network policy guidance.** Adding a
   built-in callback that POSTs to `tickerr.ai` means every LiteLLM
   proxy that enables it needs `tickerr.ai` allowlisted at the egress
   firewall — not a problem for laptop users, a real operational
   change for enterprise deployments. Should be called out in docs
   before this lands.

5. **Vendor-specific built-in callback for an unknown-traction
   service.** LiteLLM has a clear pattern: third-party observability
   integrations live in `litellm/integrations/` but they're all
   established services (Datadog, Langfuse, OpenTelemetry, Sentry,
   etc.). Adding `tickerr` — a service the PR body itself describes as
   "crowd-sourced outage radar" with no traction signal — to the
   built-in registry sets a precedent where any vendor can add their
   reporting endpoint to the LiteLLM `callbacks` whitelist by writing
   a 240-line integration. This is a maintainership / supply-chain
   surface decision that needs project-owner sign-off, not a code
   review.

6. **Smaller code-level concern:** `_extract_status_code` at
   `tickerr.py:103-112` checks `getattr(exception, "status_code",
   None)` but litellm exceptions can also expose `code` or
   `response.status_code`. Failures from underlying httpx/aiohttp
   transport may not surface `.status_code` directly. Worth fanning out
   to a couple of fallback attribute reads, otherwise this callback
   silently drops error-type classification for a meaningful slice of
   failures.

## What "merge-after" would look like

- Endpoint URL via env var (`TICKERR_REPORT_URL`).
- Activation requires an explicit env var or kwarg
  (`TICKERR_OPT_IN=1` or `litellm.callbacks = [{"name": "tickerr",
  "consent": True}]`) — *not* just being on the registry list.
- Top-of-file docstring + docs page link to Tickerr's published
  data-retention policy and call out exactly what fields are
  transmitted (the payload-shape table is currently buried in the
  source).
- Project-owner / security-review sign-off on the precedent of
  shipping an unsolicited reporter as a built-in callback.

## Verdict reasoning

`request-changes`: code is competently written and the implementation
choices (semaphore-bounded, fail-silent, narrow error-type map) are
careful. The blockers are governance/privacy/precedent, not
correctness. Once consent gating + endpoint configurability + a
documented data-retention link are in place, this becomes
`merge-after-nits` on the `_extract_status_code` fallback alone.
