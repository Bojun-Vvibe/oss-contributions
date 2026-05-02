# BerriAI/litellm PR #27025 — Fix Redis key generation to be stable across working directories

- PR: https://github.com/BerriAI/litellm/pull/27025
- Head SHA: `6be7b77513993c0656249fa1922b05f2f42a3ed8`
- Author: @mateo-berri
- Size: +32 / -1
- Status: MERGED

## Summary

`tests/_vcr_redis_persister.py:redis_key_for` used `os.path.relpath(cassette_path)` with no `start=`, so the relative path — and therefore the Redis cache key — varied with `pwd`. A test run from the repo root and the same test run from `tests/llm_translation/` produced different keys for the same cassette file, causing cache misses and replay inconsistency. Fix: pin `start=_REPO_ROOT` (computed at module load), with a `ValueError` fallback to `basename` for paths outside the repo.

## Specific references

- `tests/_vcr_redis_persister.py:16` — `_REPO_ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))` resolves the repo root once at import. `__file__` is `tests/_vcr_redis_persister.py`, so two `dirname` levels = repo root. Correct.
- `tests/_vcr_redis_persister.py:27-31` — `abs_path = os.path.abspath(...)`, then `os.path.relpath(abs_path, start=_REPO_ROOT)` inside a `try`. The `ValueError` catch handles the Windows cross-drive case (`os.path.relpath` raises when no relative path exists, e.g. `C:\foo` vs `D:\bar`). The basename fallback is the right defensive choice — it'll cause a cache miss but won't crash.
- `tests/llm_translation/test_vcr_redis_persister.py:84-108` — new `test_redis_key_is_stable_across_working_directories` asserts identical keys from three different `cwd`s (repo root, subdir, `tmp_path`) plus pins the exact expected key string. Tight assertion.

## Verdict: `merge-as-is`

Bug is real, root cause correctly identified, fix is minimal and uses the right primitive (`os.path.relpath(..., start=...)`). The test pins both the equivalence-across-cwds property *and* the concrete key string, which catches both regressions and accidental key-format drift. `_REPO_ROOT` resolution at import time is appropriate for a test helper module.

## Notes (non-blocking)

1. The cassette path passed to `test_redis_key_is_stable_across_working_directories` (`tests/llm_translation/cassettes/test_anthropic/test_streaming.yaml`) doesn't have to exist on disk — `redis_key_for` is pure path math. That's fine, but a comment to that effect would prevent future readers from "fixing" it by creating a fixture file.
2. If `_REPO_ROOT` is ever wrong (e.g. file is moved, or someone vendors `_vcr_redis_persister.py` outside `tests/`), every key silently shifts. A defensive assertion at module load — `assert os.path.basename(_REPO_ROOT)` matches an expected directory name, or the presence of a sentinel file like `pyproject.toml` — would surface that loudly.
3. The `os.path.basename(abs_path)` fallback strips the directory structure entirely, so two cassettes with the same filename in different directories would collide on cache key. Acceptable for the cross-drive edge case (rare on CI), but worth noting in a comment.
