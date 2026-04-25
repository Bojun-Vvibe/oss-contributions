# BerriAI/litellm #26469 — [WIP] Cache LiteLLM_Config param reads in DualCache and batch

- **Repo**: BerriAI/litellm
- **PR**: [#26469](https://github.com/BerriAI/litellm/pull/26469)
- **Head SHA**: `263e20fd2746c5460bc6dbd3b4ce607ca61a4016`
- **Author**: Michael-RZ-Berri
- **State**: OPEN, WIP (+246 / -10)
- **Verdict**: `needs-discussion`

## Context

The proxy's `add_deployment` housekeeping tick (every ~30s) does
4-6 serial `find_first` / `find_unique` reads against the
`litellm_config` Postgres table for the same handful of param
names: `general_settings`, `litellm_settings`,
`model_cost_map_reload_config`,
`anthropic_beta_headers_reload_config`. At 200-pod deployments
this is ~200 qps of largely-redundant point reads. PR adds a
`DualCache`-backed cache (`prisma_client._config_param_cache`)
plus a `prefetch_config_params([...])` batch prime, with
explicit invalidation after every upsert/delete.

## Design

Three layers, all in `litellm/proxy/proxy_server.py` and
`litellm/proxy/utils.py`:

1. **Cache wiring** at
   `proxy_server.py:2978-2993` plumbs the existing
   `redis_usage_cache` into `prisma_client._config_param_cache`
   so cluster pods share fills via Redis instead of each pod
   doing its own read-through.

2. **Batched warm-up** at `proxy_server.py:4980-5004`. Top of
   `add_deployment` calls
   `prisma_client.prefetch_config_params([...])` with the four
   known param names. The intent (per inline comment at
   lines 4987-4998) is one `find_many` instead of four serial
   `find_first`/`find_unique`. The four downstream call sites
   then switch from `prisma_client.db.litellm_config.find_*` to
   `prisma_client.get_generic_data(key="param_name", value=...,
   table_name="config")`, which presumably reads through the
   cache populated by the prefetch.

3. **Invalidation discipline** — every `upsert` or `delete`
   against `litellm_config` is followed by
   `await prisma_client.invalidate_config_param_cache(<key>)`.
   I count nine call sites in this diff (lines 5366, 5480,
   12685, 12873, 13157, 13523, 13596, 13652, 13885 — and
   continuing into the truncated portion of the diff for
   anthropic-beta-headers). That's the right level of
   thoroughness; if any one is missed, a single pod will serve
   stale config until TTL.

## Risks

- **Marked `[WIP]`** by the author — call out the WIP markers
  before approving. The diff itself looks coherent but the
  author hasn't claimed it's done.
- **`prefetch_config_params` and `_config_param_cache`
  implementation isn't visible in the diff I reviewed** — they
  must live in `prisma_client.py` (not `proxy/utils.py`).
  Reviewers must inspect:
  - That `prefetch_config_params` actually does one `find_many`
    (not four `find_first` calls in a loop, which would defeat
    the point).
  - That `get_generic_data(table_name="config")` consults
    `_config_param_cache` first and falls through to Postgres
    on miss.
  - That cache TTL is sane (30s? 5m? infinite-with-explicit-
    invalidate?). The PR comment at lines 4987-4998 implies
    "every TTL window," so there's *some* TTL — needs to be
    documented.
- **Nine invalidation sites is fragile.** Every future write
  path against `litellm_config` will need to remember the
  invalidate call. A wrapper like `litellm_config_upsert(k, v)`
  that does both atomically would be a safer long-term shape;
  worth at least filing a follow-up.
- **Redis sharing means a stale Redis fill can poison the whole
  cluster** until TTL. The invalidation calls only invalidate
  the local pod's view unless `invalidate_config_param_cache`
  reaches into Redis too — the implementation is not in the
  visible diff. Critical to verify before merge.
- **No test coverage in the visible diff** beyond what may be in
  the not-shown `prisma_client.py` changes —
  `tests/test_litellm/proxy/test_proxy_server.py` has +22
  lines but they're outside the visible diff slice. A test that
  proves "upsert → next read sees new value across pods" would
  be the killer test here.

## Suggestions

- **Wrap upsert+invalidate in a single helper** so callers
  cannot forget. Nine sites today, more tomorrow.
- **Document the cache TTL** in the inline comment at
  `proxy_server.py:4987-4998` — "satisfies CLAUDE.md's no-N+1
  rule" doesn't tell future readers how long stale data can
  persist.
- **Add a cluster-coherence test**: pod A upserts via
  `update_config`, pod B reads via `_check_and_reload_*`, must
  observe new value within `<TTL>`.
- Drop the `[WIP]` tag and split into "infra: add
  `_config_param_cache`" + "perf: route config reads through
  cache" if reviewers want to bisect a regression later.

## What I learned

The shape — "DualCache (in-process LRU + Redis) with explicit
invalidation on writes" — is a solid pattern for hot,
small-cardinality, write-rare config tables. The risk is
exactly what this PR's surface shows: invalidation discipline
across many call sites. Every project I've seen with this
pattern eventually grows a `Cached*Repository` wrapper to
centralize it; worth doing while the call-site count is still
nine.
