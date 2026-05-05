# BerriAI/litellm #27181 — feat(integrations): add Tickerr callback for LLM failure reporting

- **Head SHA:** `640efb1380aa73c15a5f63c34ce7772396f46502`
- **Base:** `litellm_internal_staging`
- **Author:** imviky-ctrl
- **Size:** +337 / −0 across 5 files (new integration module, registry hookup, callbacks JSON, docs)
- **Verdict:** `request-changes`

## Summary

Adds `TickerrLogger` as a built-in LiteLLM `CustomLogger` subclass that POSTs
LLM-failure metadata (provider, model, HTTP status, error type, latency) to
`https://tickerr.ai/api/v1/report` from a fire-and-forget daemon thread on
every `log_failure_event` / `async_log_failure_event`. Registered in
`_custom_logger_compatible_callbacks_literal` at `litellm/__init__.py:152` so
users can opt in with `litellm.callbacks = ["tickerr"]`.

## What's right

- **Stdlib-only.** No new pip deps. `urllib.request` for the POST,
  `threading.Thread(daemon=True)` for the fire-and-forget at
  `tickerr.py:103-122`. Won't pull `requests`/`httpx` mismatches into the
  proxy image.
- **Non-blocking by design.** `_fire_and_forget` spawns a daemon thread per
  failure with a 5-second `urlopen` timeout (`tickerr.py:118`) and a bare
  `except Exception: pass` so it cannot crash the caller. Network failure or
  Tickerr outage degrades to a no-op.
- **Provider normalization.** `_PROVIDER_MAP` at `tickerr.py:31-55` is a clean
  dict mapping LiteLLM `custom_llm_provider` values onto Tickerr slugs, with a
  fallback regex chain at `tickerr.py:69-87` that recognizes `claude*`,
  `gpt*|o[1-9]*`, `gemini*`, `mistral|mixtral`, `llama`, `command`, `grok`,
  `deepseek` model-name prefixes. Reasonable coverage for the common case.
- **Integration is additive.** Three integration points only:
  `litellm/__init__.py:152` adds `"tickerr"` to the literal type,
  `litellm/litellm_core_utils/custom_logger_registry.py:106` registers the class,
  `litellm/integrations/callback_configs.json:2-22` adds the UI config entry.
  No core code paths altered.
- **Docs are accurate to the implementation.** `docs/observability/tickerr.md`
  describes only what the code actually sends (provider, model, status,
  error_type, latency_ms — matches `_report` at `tickerr.py:170-200`).

## Concerns

### 1. Privacy / outbound-egress: this is the wrong default surface for a
default-merged callback

The PR positions Tickerr as a callback users opt into via
`litellm.callbacks = ["tickerr"]`. That's true today — but landing it as a
**built-in registered callback** (lines `litellm/__init__.py:152` and
`custom_logger_registry.py:106`) means it ships *in the wheel and the proxy
image* by default. The class is dormant until a user mentions it by name, but
the import side-effects are not zero-risk:

- The `custom_logger_registry.CustomLoggerRegistry` dict lookup imports
  `TickerrLogger` at module-load time
  (`custom_logger_registry.py:48 from litellm.integrations.tickerr import TickerrLogger`).
  Import-time side effects are minimal here (just the env-var reads in
  `__init__`), but the precedent matters: every built-in callback added this
  way ships executable HTTP-emitting code into every LiteLLM install.

- Tickerr is a **third-party service** (`tickerr.ai`) outside the BerriAI/LiteLLM
  org. By landing it as a registered name rather than a separate package
  (`pip install litellm-tickerr`), LiteLLM users implicitly trust the BerriAI
  release process to vouch for what `https://tickerr.ai/api/v1/report` does
  with the data. There is no SECURITY.md cross-link, no domain-pinning, no
  mention of the data-retention policy, no contractual relationship surfaced
  in this PR.

  **Recommendation:** Either (a) move `TickerrLogger` to a separate package
  (`litellm-tickerr`) installed only by users who explicitly want it — same
  pattern many observability backends use — or (b) keep it built-in but add a
  `SECURITY.md`-level note in `docs/observability/tickerr.md` covering data
  retention, a link to the Tickerr privacy policy, and BerriAI's vetting
  posture.

### 2. The "anonymous" claim is partially-correct at best

`docs/observability/tickerr.md:11` states "Anonymous. Zero overhead on success
paths." and the README repeats "No API key required. Anonymous." However:

- `provider`, `model`, `error_code`, `latency_ms` are sent as a tuple. In a
  proxy deployment serving a known model mix to a known account, the source
  IP of the egress request (which the receiving server logs by default) plus
  the model fingerprint is enough to deanonymize the reporter to anyone with
  Tickerr access. The PR's own example payload at `tickerr.py:191-199`
  confirms `region` and `client_tier` are also opt-in joinable to a tenant
  identity.
- There is no explicit IP-stripping on the receive side promised in this PR.
- The `User-Agent` header is `litellm-tickerr/1.0` (`tickerr.py:23`) which is
  a clean fingerprint for traffic-classification at the receive side.

The honest claim is "no API key, no prompts, no responses, no LiteLLM-issued
identifiers" — not "anonymous." The doc copy at `docs/observability/tickerr.md:11`
should be softened to match.

### 3. No tests

The PR template explicitly calls out: *"Adding at least 1 test is a hard
requirement"* (PR body line `[ ] I have Added testing in tests/test_litellm/`).
The PR includes zero tests. At minimum:

- A unit test that `_normalize_provider("claude-3-5-haiku", {})` →
  `"anthropic"` and that `_PROVIDER_MAP` fallthrough works.
- A unit test that `_extract_status_code(exception)` handles `int`, str-int,
  None, missing attribute.
- A unit test that mocks `urllib.request.urlopen` and confirms `_fire_and_forget`
  posts the right JSON shape and never raises.
- A test that the registry lookup `CustomLoggerRegistry["tickerr"]` returns
  `TickerrLogger`.

Without any of these, regressions in `_PROVIDER_MAP` (a hand-maintained dict
of 22 entries) or in payload shape will silently land.

### 4. `log_success_event` is not implemented despite the doc claim of
"recovery signals"

`docs/observability/tickerr.md:75-79` describes a `recovering` state driven by
"Reports dropping, recovery signals arriving." The class only implements
`log_failure_event` / `async_log_failure_event` (`tickerr.py:139-160`). There
is no `log_success_event` that would emit `is_resolution: true` (the field is
threaded through `_report`'s `is_resolution: bool` parameter at line 167 but
the only call sites pass `is_resolution=False`). Either implement the success
path that emits resolution events, or remove the recovery-state copy from the
docs to match the implementation.

### 5. Provider regex order at `tickerr.py:78-80` has a subtle bug

```python
if re.match(r"^gpt|^o[1-9]", model, re.I):
    return "openai"
```

The pattern `^o[1-9]` matches `o1`, `o2`, …, `o9` reasoning-model prefixes —
but it also matches any model name beginning with `o` followed by a digit,
e.g. a hypothetical `oracle-1`-prefixed model would be misclassified as
`openai`. Tighten to `^o[1-9](?:[a-z-]|$)` or pin to known prefixes. Low
severity, but the kind of thing tests would have caught.

### 6. Minor

- `tickerr.py:117-118`: `urlopen(req, timeout=5)` — 5s is reasonable but
  consider making it configurable via `TICKERR_TIMEOUT` env to match
  `TICKERR_CLIENT_TIER` and `TICKERR_REGION`.
- `tickerr.py:135`: `__init__(self, **kwargs)` reads env vars once at instance
  construction time. If a long-running proxy has `TICKERR_REGION` rotated
  (e.g. region migration), the change won't be picked up until the proxy
  restarts. Acceptable but worth a doc note.
- `_ERROR_TYPE_MAP[500]` → `"overloaded"` (`tickerr.py:58`) is a generous
  classification; many 500s are not capacity-related (genuine bugs, malformed
  responses). Consider mapping 500 → `"server_error"` distinct from 503/529 →
  `"overloaded"`.
- The fallback `error_type = ... if status_code else None` at
  `tickerr.py:184` returns `None` if the exception had no status code at all
  (network errors, timeouts on the LiteLLM side, JSON-parse failures). These
  are arguably the most interesting failures from an outage-radar standpoint
  and currently get dropped from `error_type`.

## Verdict rationale

The integration code itself is competent (stdlib-only, fire-and-forget,
non-blocking, additive). The `request-changes` verdict is driven by:
(a) the architectural question of whether a third-party reporting service
should ship as a built-in registered callback or as a separate package
(concern 1); (b) the missing tests that the PR template flagged as a hard
requirement (concern 3); (c) the docs/implementation mismatch on recovery
signals (concern 4); and (d) the partially-correct "anonymous" claim that
should be softened before users adopt it (concern 2).

Once (1) is decided (built-in vs. separate package), tests are added, and
docs are aligned with implementation, this is a clean addition.
