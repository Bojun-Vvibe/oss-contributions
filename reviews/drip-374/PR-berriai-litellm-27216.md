# BerriAI/litellm #27216 — Include model name + configured TPM/RPM in priority rate-limit 429 errors

- URL: https://github.com/BerriAI/litellm/pull/27216
- Head SHA: `4d84107f9aaac7398c80a940560375d99a45f1b9` (`4d84107`)
- Author: @ishaan-berri
- Diff: +100 / −0 across 2 files
- Verdict: **merge-as-is**

## What this changes

A 5-line operator-facing diagnostics improvement plus a 95-line
regression test. In `litellm/proxy/hooks/dynamic_rate_limiter_v3.py`
the two HTTP 429 detail strings emitted by `_check_rate_limits` —
one for the model-capacity-reached branch (line 498-503) and one for
the priority-based limit-exceeded branch (line 517-528) — gain
`Model: {model}`, `Model TPM: {model_group_info.tpm}`, and
`Model RPM: {model_group_info.rpm}` interpolated fields. No control-flow
changes, no schema changes, no new imports.

The motivating UX problem is captured precisely in the PR body: the
prior message was

> Priority-based rate limit exceeded. Priority: prod, Rate limit type:
> tokens, Remaining: -664145, Model saturation: 86.3%

When you're operating a multi-model multi-priority deployment, that
message tells the on-call engineer that *some* request hit *some*
limit at 86.3% saturation — but not which model, and not what the
configured limit was, so they can't even start to answer the actual
operator question: "is this priority class under-allocated, or is the
model TPM just too small for current traffic?" After this PR the
message becomes self-diagnosing — model name + configured TPM/RPM are
in-line, so the operator can compare "saturation vs. configured TPM"
without having to cross-reference router config or Prometheus.

## The test is the load-bearing piece

The new
`test_priority_429_includes_model_name_and_configured_limits` at
`tests/test_litellm/proxy/hooks/test_dynamic_rate_limiter_v3.py:1681-1773`
sets up a real `Router` with a single model entry pinning
`tpm=1_000_000` and `rpm=10_000`, fetches the
`model_group_info` via `handler.llm_router.get_model_group_info(...)`
(the same helper the production path uses), then patches
`v3_limiter.atomic_check_and_increment_by_n` to return a synthetic
`OVER_LIMIT` response so the saturated-priority branch is reached
without needing actual Redis traffic. The assertions at lines 1764-1773
pin both the new fields (`Model: gpt-4o-test`, `Model TPM: 1000000`,
`Model RPM: 10000`) AND the existing fields (`Priority: prod`,
`Rate limit type: tokens`, `Model saturation:`) so future refactors
can't silently drop either set.

`saturation=0.95` is deliberately above whatever the priority-enforce
threshold is, and `over_limit_response.statuses[0]` is shaped with
`code=OVER_LIMIT, descriptor_key=priority_model, rate_limit_type=tokens`
to land specifically in the priority-based 429 branch (not the model-
capacity 429 branch). Both branches got the new fields and the test
covers the priority-based one, which is the one the PR body
explicitly targets; the model-capacity branch change is symmetric and
trivial enough that the lack of a dedicated test for it is acceptable.

## Concerns

(a) `model_group_info.tpm` and `model_group_info.rpm` are interpolated
without a None-guard. If a model_group exists but has no `tpm`/`rpm`
configured (which the production path shouldn't reach because
`_check_rate_limits` is only called when limits are configured, but
which a future refactor could violate), the formatted message would
read `Model TPM: None, Model RPM: None`. That's not crash-causing
and is arguably the right behavior (operator sees "no limits
configured" explicitly), but worth noting in the merge thread so
nobody later "fixes" it by suppressing the field — `None` here is a
genuine signal.

(b) The 95-line test sets `os.environ["LITELLM_LICENSE"] = "test-license-key"`
at module-level inside the test (line 1690). If that env var leaks
across tests in the same pytest worker, downstream tests asserting
license-not-set behavior could go green incorrectly. Not a regression
this PR introduces (looks like the surrounding tests do this too) but
worth a follow-up to wrap in `monkeypatch.setenv` / `with
mock.patch.dict(os.environ, ...)` for hermeticity.

(c) The error-message format is now load-bearing for any operator
log-parsing / alerting infra that greps the 429 detail. That's a
trivial breaking change for anyone with alert rules keyed on the old
text, but the change is strictly additive (existing fields preserved
in their existing order, new fields inserted between them), so any
substring-match alert on `"Priority-based rate limit exceeded"` or
`"Priority: "` continues to fire. Worth calling out in the changelog.

Pure additive operator-DX win, well-tested, narrow scope. Ships.
