# BerriAI/litellm#26967 — tests(vcr): instrument Redis target + per-cassette persist log lines

- **PR**: https://github.com/BerriAI/litellm/pull/26967
- **Head SHA**: `3a27a4fdbf27e44a7cf0eade411f9a6a3837c738`
- **Size**: +77 / -2, 2 files
- **Verdict**: **merge-as-is**

## Context

litellm's test stack uses VCR with a Redis-backed cassette persister so CI workers share recorded HTTP interactions. When this misbehaves (worker pointed at wrong Redis, hits when there should be misses, evictions) there is currently nothing in the CI log to triage from — you see test failures but not why VCR served (or didn't serve) the cassette. This PR adds tagged log lines around every cassette load/save and a once-per-worker startup banner with Redis target + memory metrics.

## Design analysis

### Logger bootstrap (`tests/_vcr_redis_persister.py:13-20`)

```python
_log = logging.getLogger("litellm.vcr.persister")
if not _log.handlers:
    _h = logging.StreamHandler()
    _h.setFormatter(logging.Formatter("[VCR] %(message)s"))
    _log.addHandler(_h)
    _log.setLevel(logging.INFO)
    _log.propagate = False
```

Idempotent installer (the `if not _log.handlers` guard prevents duplicate-handler-on-reimport, which is real for pytest-xdist worker reload paths). `propagate=False` keeps these out of the root logger so they don't double-print under pytest's `--log-cli-level`. Tagged with `[VCR]` prefix exactly so `grep '\[VCR\]' build.log` works — the comment at `:13` even calls that out.

### Per-cassette events (`:73-92`)

`load_cassette`:
- miss → `_log.info(f"miss key={key}")` then raise `CassetteNotFoundError()`
- hit → `_log.info(f"hit  key={key} bytes={len(data)}")` (note the alignment space after `hit` to line up with `miss` — small nicety for log skim)

`save_cassette`:
- success → `_log.info(f"persist key={key} bytes={len(payload)} episodes={req_count}")`
- exception → `_log.info(f"persist-failed key={key} bytes={len(payload)} episodes={req_count} err={type(exc).__name__}: {exc}")` then re-raise.

The exception branch logs *and re-raises* — preserves the original failure semantics, just adds visibility. `episodes={req_count}` is the cassette's recorded request count, useful for spotting cassettes that should have N episodes but persisted with 0.

### Startup banner (`:98-148`)

`log_redis_target_banner()` — designed to run once per pytest worker:
1. Resolve the URL via `_redis_url_from_env()`. If none → `_log.info("VCR disabled (no REDIS_URL/REDIS_SSL_URL/REDIS_HOST in env)")` and return. Important: the banner *also* runs in the `_vcr_disabled()` branch (see conftest hook below) so you always know whether the worker decided to use VCR.
2. **Credential redaction** at `:108-113`: `rediss://user:pass@host:port` → `rediss://user:***@host:port`. Splits on `@` once, then on `:` once from the right of the user-info, replaces password segment. Conservative — only redacts when both `@` and `:` are present in the user-info portion.
3. Connect with `socket_timeout=5`, `socket_connect_timeout=5`, `decode_responses=True`. The 5s timeouts are the right choice for a banner that should never hang the test session.
4. Log `info("memory")` keys: `maxmemory_human`, `maxmemory_policy`, `used_memory_human`, `used_memory_peak_human`. These are exactly the four that matter for "is Redis about to evict our cassettes" triage. `maxmemory_policy=allkeys-lru` with growing `used_memory` is the canonical "your hits will start missing" pattern.
5. Log `info("stats").get("evicted_keys")` — wrapped in its own try because some Redis variants don't expose `stats`.
6. `client.scan_iter(match=f"{REDIS_KEY_PREFIX}*", count=500)` to count existing VCR keys. `count=500` is the SCAN cursor batch size, not a limit — full prefix is enumerated. This bounds wall-clock for very large prefixes (10k+ keys) to a handful of round-trips.
7. Final `except Exception as exc: _log.info(f"banner failed: ...")` — banner *never* breaks the test session even if Redis is down.

### Conftest hook (`tests/llm_translation/conftest.py:120-127`)

```python
def pytest_recording_configure(config, vcr):
    if _vcr_disabled():
        log_redis_target_banner()
        return
    log_redis_target_banner()
    vcr.register_persister(make_redis_persister())
    patch_vcrpy_aiohttp_record_path()
```

Banner fires in *both* arms — even if VCR is disabled, the banner prints (and prints "VCR disabled (...)") so you know which mode the worker is in. Symmetric and correct.

## What's right

- **Tagged prefix `[VCR]`** so the noise floor is greppable in CI logs.
- **`propagate=False`** so these don't double-print under pytest's CLI logger setup.
- **Idempotent handler install** — important for xdist worker reload.
- **Credential redaction** before logging the URL.
- **5s socket timeouts** on the banner Redis client so a misconfigured Redis can't hang the test session.
- **Banner in both VCR-on and VCR-off branches** — you always know the mode.
- **Exception-path `persist-failed`** logs and re-raises — visibility without changing semantics.
- **`scan_iter` with `count=500`** rather than `KEYS *` (which would block Redis on large prefixes).
- **Per-event `bytes=...`** on hits/persists — the canonical signal for "cassette grew unexpectedly between runs".

## Nits

- `_log.info` for failure cases (miss, persist-failed, banner failed) — arguably warning-level, but `info` is fine given the `[VCR]` prefix and the explicit grep workflow.
- The credential-redaction logic only handles the `scheme://user:pass@host` shape. A bare-password URL like `redis://:pass@host` (no user) goes through the same code path: `head = "redis://:pass"`, `scheme_user, _ = head.rsplit(":", 1)` → `scheme_user = "redis://"`, output `"redis://:***@host:port"`. Works correctly. A URL with no `@` (no creds) skips redaction. ✓
- `_log.setLevel(logging.INFO)` is hard-coded; a `LITELLM_VCR_LOG_LEVEL` env override would let users dial it down to WARNING in noisy CI shards. Not blocking.
- The `episodes={req_count}` counter uses `cassette_dict.get("requests", []) or []` — handles both missing key and `None` value. ✓
- No test arm for the new helpers. Acceptable — these are themselves test infrastructure, and the round-trip through `make_redis_persister` is exercised by every test that uses VCR.

## Verdict

**merge-as-is** — clean, conservative test-infra observability addition. Idempotent installer, credential-safe, bounded latency, fail-quiet. The kind of instrumentation PR that pays for itself the first time CI flakes on a Redis issue. No correctness risk to production code (test-only).
