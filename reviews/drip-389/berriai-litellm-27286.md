# BerriAI/litellm#27286 — fix(logging): override phoenix project name (per-project TracerProvider LRU)

- Head SHA: `05b8b9a150296f0d0232de4ea471ad0dd7fb2f36`
- Author: @mubashir1osmani
- Link: https://github.com/BerriAI/litellm/pull/27286

## Notes
- `litellm/integrations/arize/arize_phoenix.py:40` introduces `_MAX_PROJECT_PROVIDERS = 64` with the comment that "Evicted providers are NOT shut down — in-flight spans must not be interrupted." That's defensible, but it leaks `BatchSpanProcessor`s and exporter sockets indefinitely. At 64 high-churn projects this may be tolerable; document the upper bound on resource usage.
- `_init_tracing` at `arize_phoenix.py:73–88` builds a `default_provider` eagerly and seeds the LRU. Good — avoids a cold-path branch on first call.
- `dev_config.yaml:12` adds `callbacks: ["arize_phoenix"]` — fine for dev, but make sure CI doesn't pick this up and start emitting spans to a real endpoint.
- Use of `OrderedDict` is fine; ensure `_get_or_create_project_provider` uses `move_to_end` on hit (not visible in the 120-line excerpt — please verify) for true LRU semantics.

## Verdict
`merge-after-nits`

Approach is right — per-project provider is the only OTEL-clean way to route. Verify LRU `move_to_end` on cache hit, and add a metric/log when an eviction happens so operators can spot churn.
