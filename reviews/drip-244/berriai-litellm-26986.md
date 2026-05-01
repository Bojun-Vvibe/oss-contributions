# PR #26986 — tests/ci: cassette proxy + opt every cost-bearing CI job in

- Repo: BerriAI/litellm
- Head: `a9842cd3a370122ee3db90ffa0da45f223fb8f4e`
- URL: https://github.com/BerriAI/litellm/pull/26986
- State: DRAFT
- Verdict: **merge-after-nits**

## What lands

+1544 / −0. Builds on prior VCR cassette work (#26838) which only covered
in-process pytest jobs via httpx/aiohttp/httpcore monkey-patching. This PR
adds a **network-level recording HTTP/HTTPS proxy** so the in-Docker e2e jobs
(`agent_testing`, `audio_testing`, `image_gen_testing`, `logging_testing`,
`ocr_testing`, `search_testing`) — which vcrpy structurally cannot reach —
also stop hitting upstream provider APIs on every CI run.

Architecture: pytest/SUT process → `HTTPS_PROXY=cassette-proxy:8080` →
mitmproxy + addon → on miss forwards upstream and stores `(request, response)`
to Redis as MessagePack; on hit replays from Redis without any upstream
traffic. Localhost / `host.docker.internal` / proxy itself are passthrough.

Components:

- `tests/e2e_cassette_proxy/cache_key.py` — pure-function key derivation:
  hashes `(method, scheme, host, path, sorted query, allowlisted headers,
  canonical-JSON body)` with auth/tracing/SDK-metadata headers stripped so
  equivalent requests collide regardless of run-to-run noise.
- `tests/e2e_cassette_proxy/redis_store.py` — one `(request, response)` pair
  per key; per-key payload size cap; oversize responses dropped with a log
  line; Redis errors never block the request path.
- `tests/e2e_cassette_proxy/addon.py` — mitmproxy addon. Non-2xx upstream
  responses are not persisted (good — don't cache transient 5xxs).
- `tests/e2e_cassette_proxy/Dockerfile` — fully pinned fallback image.
- `tests/e2e_cassette_proxy/trust_ca.sh` — adds proxy CA to every Python /
  curl / boto3 / node trust store.

Three reusable CircleCI commands at `.circleci/config.yml:114-300`:

- `start_cassette_proxy` — installs mitmproxy via `uv tool install` and runs
  `mitmdump` as a background subprocess (works on both `docker:` and
  `machine:` executors). Exports `CASSETTE_PROXY_URL` (containers via
  `host.docker.internal:8080`), `CASSETTE_PROXY_HOST_URL` (runner shell via
  `localhost:8080`), and `CASSETTE_PROXY_CA` into `$BASH_ENV`.
- `export_cassette_proxy_docker_args` — composes a single
  `$CASSETTE_PROXY_DOCKER_ARGS` string of `-e ...` / `-v ...` flags ready to
  splice into a SUT container's `docker run`. Two-line opt-in for in-Docker
  jobs.
- `enable_cassette_proxy_for_pytest` — patches certifi's bundled `cacert.pem`
  in every venv on the runner so openai-python / httpx / langfuse /
  google-auth trust the proxy CA without code changes; exports `HTTPS_PROXY` /
  `NO_PROXY` / `SSL_CERT_FILE` for every subsequent step; **flips
  `AIOHTTP_TRUST_ENV=true`** — without that flag litellm's aiohttp transport
  silently ignores `HTTPS_PROXY` (PR body cites
  `litellm/llms/custom_httpx/http_handler.py:951-952`).

## What's good

- The `AIOHTTP_TRUST_ENV=true` callout is exactly the kind of load-bearing
  gotcha that costs other PRs days. Documenting it inline in two places
  (docker args + pytest opt-in) is correct.
- `NO_PROXY` includes `localhost,127.0.0.1,0.0.0.0,host.docker.internal,
  postgres-db,redis-cache,cassette-proxy` plus dynamic `${REDIS_HOST}` —
  Redis isn't HTTP, mitmproxy can't proxy it, and the explicit sidecar
  hostnames prevent surprise loops.
- Resolving Redis target from `REDIS_SSL_URL` → `REDIS_URL` →
  `REDIS_HOST/PORT/PASSWORD` triple at `start_cassette_proxy` is the right
  precedence order for litellm's existing env conventions.
- `--set ssl_insecure=true` on mitmdump is acceptable for a recording proxy
  (CI traffic only, no production secrets), but worth a one-line comment so a
  future reader doesn't think it's a security oversight.
- Stripping auth/tracing/SDK-metadata headers from the cache key is the only
  way per-run noise (e.g. `traceparent`, `x-request-id`, `User-Agent`
  containing dynamic SDK versions) doesn't tank cache hit rate.

## Nits

- The CA-fetch loop at `start_cassette_proxy` uses `for i in 1 2 3 4 5; do
  curl … || true; sleep 2; done` then `test -s /tmp/cassette-proxy-ca.crt`.
  If all five attempts fail the `test -s` correctly fails the step, but the
  five `sleep 2` adds 10 s tail latency on slow runner cold-starts where
  mitmdump took longer than the `wait_for_service` 60 s. Consider a single
  longer-timeout fetch with explicit error message ("`mitm.it/cert/pem` not
  reachable — is mitmdump actually listening?") to improve debug surface.
- `find / -name cacert.pem 2>/dev/null` in `enable_cassette_proxy_for_pytest`
  walks the entire root filesystem — slow and noisy. Scope to
  `~/.cache/uv`, `~/.local/share`, `/usr/lib/python*`, `/opt/circleci/`
  with `find <dir> -name cacert.pem`. Saves ~5-15 s per CI step.
- `grep -q -F "$(head -n 2 "$CASSETTE_PROXY_CA")" "$cacert"` is a fragile
  idempotency check (matches only the first two lines, which are nearly
  always `-----BEGIN CERTIFICATE-----` plus base64 first row). Better: write
  a sentinel comment line `# cassette-proxy-ca-installed-${CIRCLE_SHA1}` at
  append time and grep for that.
- The PR description mentions `tests/e2e_cassette_proxy/Dockerfile` is "kept
  as a fallback for environments that don't have `uv tool install`
  available" — but the three CircleCI commands all assume `uv` is present
  (via prior `install_uv` step). Worth a one-line README note clarifying
  *which* environments take the Dockerfile path so the dual-implementation
  surface doesn't silently bit-rot.
- No mention of redaction policy for *response* bodies. Cache keys strip auth
  from requests, but cassette responses likely contain provider-issued tokens
  in error messages, rate-limit headers carrying account info, etc. Consider
  documenting (or implementing) a response-body redactor in `addon.py` before
  this is enabled in shared CI Redis.
- DRAFT state suggests author isn't done. Confirm the `.circleci/config.yml`
  job-level integrations downstream of these three commands actually compile
  before un-drafting.

## Why merge-after-nits not merge-as-is

The fundamental architecture is solid and the gotchas (`AIOHTTP_TRUST_ENV`,
`NO_PROXY` for Redis, non-2xx not persisted) show real understanding of where
this stuff bites. The nits are operational hygiene (find scoping, idempotency
sentinel, response redaction) — none of them block merge but the redaction
question deserves an explicit answer before this is enabled against a
production-shared Redis instance.