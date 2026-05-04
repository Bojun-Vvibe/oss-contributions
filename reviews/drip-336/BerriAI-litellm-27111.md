# BerriAI/litellm #27111 — fix(MLP-6153): reload router cache on /model/info miss to fix Terraform race condition

- **Head SHA reviewed:** `47b47620e8959a23503a4fc71cc04b780632b97c`
- **Size:** +183 / -0 across 2 files
- **Verdict:** merge-after-nits

## Summary

Fixes a flaky Terraform read-after-apply for `litellm_model` resources
in multi-replica deployments. When `model_info_v1` is asked for a
specific `litellm_model_id`, a cache miss on
`llm_router.get_deployment(model_id=…)` previously returned a 400
immediately. With this PR, on miss the handler calls
`proxy_config.add_deployment(prisma_client=…, proxy_logging_obj=…)`
to reload the router cache from the DB, then retries the lookup once
before raising 400.

171-line test file covers the warm path, the miss-then-hit path, and
the still-missing path.

## What I checked

- `litellm/proxy/proxy_server.py:11185-11195` (added block) — guard
  is `deployment_info is None and prisma_client is not None`, which
  correctly avoids the reload attempt in non-DB modes (config-only
  deployments). The `try/except: pass` catches reload failures
  silently and falls through to the existing 400 raise; that's the
  right behavior for a single retry, though a `verbose_proxy_logger`
  warn would help operators correlate spurious 400s with reload
  failures rather than DB outages.
- `tests/proxy_unit_tests/test_model_info_cache_reload.py:51-78`
  (warm-cache test) — patches module globals via `setattr(proxy_server, …)`
  and asserts `add_deployment.assert_not_called()`. Correct.
- `test_model_info_cache_reload.py:81-110` (regression test) —
  `mock_router.get_deployment.side_effect = [None, deployment]` is
  the precise reproduction of the race; assert on
  `call_count == 2` confirms exactly one retry. Excellent test.
- `test_model_info_cache_reload.py:113+` (still-missing test, per PR
  body) — confirms 400 is raised when the model is genuinely absent
  even after reload.

## Concerns / nits

1. **Pod thrash under sustained 404s.** If a client polls
   `/model/info?litellm_model_id=does-not-exist` in a hot loop, every
   request now triggers a full `proxy_config.add_deployment(...)`
   reload — that hits Prisma, walks the entire model table, and
   re-applies router state. On a large deployment this is a real cost
   and a minor amplification vector. Consider rate-limiting reloads
   per process (e.g. coalesce within a 1–5 s window) or a small
   negative cache to bound the blast radius.
2. **Silent `except Exception:`.** The bare except will swallow
   `asyncio.CancelledError` (Python 3.7+ makes it a `BaseException`,
   so technically safe, but worth narrowing) and any genuine DB
   errors. Replace with `except Exception as e:` and at minimum
   `verbose_proxy_logger.exception("model/info cache reload failed", …)`.
3. **No comment in source.** The 12-line block has a 4-line inline
   comment that's good, but the *why* (multi-replica eventual
   consistency on Read-after-Apply) is captured in the test docstring
   only. A one-line link to the Linear ticket / PR number above the
   added block would help future readers grep for context.
4. The added test relies on monkey-patching module globals
   (`setattr(proxy_server, "llm_router", …)`) without restoring them
   in a teardown. If pytest runs other tests that import
   `proxy_server` after these run, they'll see the mock. A
   `pytest.fixture(autouse=True)` with a `yield`-then-restore pattern
   would isolate this.

## Risk

Low-medium. The change is bounded (single retry, only when DB-backed,
only on a specific endpoint), but the silent swallow + lack of
rate-limiting could mask real DB issues or amplify load under
pathological clients.

## Recommendation

Merge after addressing nits 2 (log the swallowed exception) and 4
(test isolation). Nit 1 (rate-limit reloads) is worth a follow-up
issue but shouldn't block this fix — the race it solves is a real,
user-visible Terraform failure.
