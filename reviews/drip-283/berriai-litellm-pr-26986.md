# BerriAI/litellm PR #26986 — tests/ci: cassette proxy + opt every cost-bearing CI job in

- Head SHA: `eb6534daa41f7db677240f739912217315ce1b05`
- Size: +1923 / -18, 13 files

## Specific refs

- `tests/e2e_cassette_proxy/addon.py` (+253) — mitmproxy addon. Passthrough hosts `localhost`, `host.docker.internal`, the proxy host itself, and any `LITELLM_E2E_CASS_PASSTHROUGH_HOSTS` are never cached. Non-2xx upstream responses are not persisted (avoids poisoning cassettes with transient 429/5xx).
- `tests/e2e_cassette_proxy/cache_key.py` (+158) — pure-function cache key over `(method, scheme, host, path, sorted query, allowlisted headers, canonical-JSON body)`. Strips auth/tracing/SDK-metadata headers. 14 unit tests pin equivalence classes.
- `tests/e2e_cassette_proxy/redis_store.py` (+186) — one Redis key per `(request, response)` pair, MessagePack with JSON+base64 fallback. `max_payload_bytes` ceiling drops oversize responses with a log line. Critically: never blocks the request path on Redis errors (open-circuit fallback to upstream).
- `.circleci/config.yml:114-364` — three reusable commands (`start_cassette_proxy`, `export_cassette_proxy_docker_args`, `enable_cassette_proxy_for_pytest`). The proxy runs as a background subprocess via `uv tool install mitmproxy==11.0.2` so it works on both `docker:` and `machine:` executors without a docker daemon.
- `.circleci/config.yml` — wires 8 in-Docker SUT jobs (Pattern A) and 17 in-process pytest jobs (Pattern B). PR body lists each.
- Cassette Redis is **deliberately separate** from the project's shared Redis (`CASSETTE_REDIS_URL` secret) — comment at config.yml ~line 167 cites observed 4150 `FLUSHALL`s / 33 days on shared Redis nuking 740+ entries each time. This isolation is the single most important design decision and it's documented in-place.

## Assessment

This is a substantial, well-justified piece of CI infrastructure. The transport-agnostic mitmproxy approach is the correct response to vcrpy's structural inability to attach to Docker-isolated SUT processes — the diagnosis in the PR body matches the architecture. Per-key one-pair Redis layout sidesteps every footgun the prior episode-list cassette format hit (unbounded growth, order sensitivity, OOM under `noeviction`).

Strong points: (1) `enable_cassette_proxy_for_pytest` flips `AIOHTTP_TRUST_ENV=true` — without this the aiohttp transport silently ignores `HTTPS_PROXY` (cited at `litellm/llms/custom_httpx/http_handler.py:951-952`), which is exactly the kind of thing that would silently eat real billable calls; (2) `NO_PROXY` automatically picks up `$REDIS_HOST` so the project's managed Redis isn't proxied; (3) 31 hermetic unit tests (`test_cache_key.py`, `test_redis_store.py`, `test_addon.py`) cover the addon end-to-end including replay-only / record-only modes.

Concerns worth raising before merge:
- The cache key strips auth headers — this is correct for cross-test sharing but means a request that legitimately differs only by `Authorization` (e.g. multi-tenant routing) collides. Worth an explicit note in the README that this is by design.
- `LITELLM_E2E_CASS_REPLAY_ONLY=1` returning `599` on miss is a nice mode for proving cache completeness, but no CI job currently exercises it — adding one nightly job in replay-only mode would catch cassette drift early.
- The mitmproxy CA fetched via `http://mitm.it/cert/pem` requires the proxy to actually be reachable; the 5-attempt curl loop is fine but should bound total time (currently up to 10s of sleeps + 5×30s curl timeouts).

verdict: merge-after-nits
