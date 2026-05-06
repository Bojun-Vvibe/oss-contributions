# BerriAI/litellm#27272 — Cap Prometheus end-user metric cardinality

- PR: https://github.com/BerriAI/litellm/pull/27272
- Head SHA: `bede81b2b937c5ea14b37b4f26c957b23d31b4b0`
- Size: +356/-23
- Verdict: **merge-after-nits** (man)

## Shape

Closes the long-standing memory-leak / scrape-explosion footgun where
`prometheus_client` accumulates one label-set per unique `end_user` value
indefinitely (any proxy taking traffic from N user-id-bearing clients
eventually hits `O(N × metric_count)` retained label tuples in process
memory). New `BoundedPrometheusSeriesTracker` is constructed at
`prometheus.py:87` and all `inc_labeled_*`/`set_labeled_*` paths route through
new helper `_get_labeled_metric(metric, metric_name, labels)` at
`prometheus.py:991-1000` (replacing direct `metric.labels(**_labels).inc/set`
at `:1053` and the gauge equivalents).

The tracker is bounded by three new top-level config knobs declared at
`litellm/__init__.py:417-419`:

- `prometheus_end_user_metrics_max_series_per_metric: int = 10000`
- `prometheus_end_user_metrics_ttl_seconds: float = 3600.0`
- `prometheus_end_user_metrics_cleanup_interval_seconds: float = 60.0`

`_track_bounded_prometheus_metric_series` at `:1001-1037` short-circuits cleanly
when `END_USER` isn't a labelname for the metric (correct — the bound only
applies to the cardinality-explosion-prone metrics), and again when
`end_user is None` (also correct — only-end-user-bearing label tuples get
tracked).

## Notable observations

- The `getattr(litellm, "prometheus_end_user_metrics_max_series_per_metric", 10000)`
  pattern at `:1014-1023` is correct for forward-compat with users on older
  config files — those without the attribute fall back to the safe default.
  But the same defaults are duplicated in `__init__.py` and in this `getattr`
  fallback; if someone bumps one without the other, behavior diverges silently.
  Extract to a module-level constant `_DEFAULT_BOUNDED_PROM_*` that both refer
  to.
- Lock acquisition in `_get_labeled_metric` at `:995` (`with
  self._bounded_prometheus_series_tracker.lock`) wraps the `metric.labels(...)`
  call too — that's intentional to make the "create-or-fetch + track" pair
  atomic, but `prometheus_client.MetricFamily.labels()` itself takes its own
  internal lock. Verify there's no lock-ordering inversion possible if the
  tracker ever calls back into `metric.remove(...)`.
- `if max_series is None and ttl_seconds is None: return` at `:1029` — good
  fast-path for the "user opted out of bounding entirely" case. Document this
  in the new config knobs' docstrings (set either to `None` to disable).

## Concerns / nits

- No test in the visible diff exercising the eviction path (10001st unique
  end_user: oldest series is dropped) or the TTL path (series older than
  `ttl_seconds` is dropped on next cleanup tick). These are the two contracts
  that distinguish this PR from a noop wrapper — they need pinning tests.
- `BoundedPrometheusSeriesTracker` lives at
  `litellm/integrations/prometheus_helpers/bounded_prometheus_series_tracker.py`
  (per the import at `prometheus.py:28-30`) — that file is not in the
  first 250 lines of the diff, so I can't verify its eviction policy (LRU vs
  FIFO vs random) or how `cleanup_interval_seconds` is enforced (background
  thread? lazy on next track?). The PR body should call out the chosen
  strategy explicitly.
- The new `getattr(...)` triple at `:1014-1023` is awkward; consider a single
  `tracker_config_for(litellm)` accessor on the tracker itself.
