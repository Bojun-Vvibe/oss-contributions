# BerriAI/litellm PR #26447 — New Relic AI Monitoring integration

- **Repo:** BerriAI/litellm
- **PR:** [#26447](https://github.com/BerriAI/litellm/pull/26447)
- **Head SHA:** `faae9ad2c0d3a2badcdd74d430477b8c49922a2d`
- **Author:** bonczj
- **Size:** +2174 / −2 across 17 files
- **Reviewer:** Bojun (drip-27)

## Summary

Adds a New Relic AI Monitoring custom logger plus a `USE_NEWRELIC=true`
container entrypoint mode that runs the proxy under `newrelic-admin`,
mirroring the existing `USE_DDTRACE` pattern. New module is
`litellm/integrations/newrelic/newrelic.py` (`NewRelicLogger`), wired
through the `custom_logger_registry` and exposed via
`litellm.newrelic_params: NewRelicInitParams`. Adds docs, dashboard
asset, dashboard logo, and a `tests/test_litellm/integrations/newrelic/`
suite.

## Key changes

### `docker/prod_entrypoint.sh:9–14`

Adds an `elif [ "$USE_NEWRELIC" = "true" ]; then exec newrelic-admin
run-program litellm "$@"` branch parallel to the existing `USE_DDTRACE`
branch. Order is `DDTRACE → NEWRELIC → plain litellm`, so a deployment
setting both flags will run under DDTRACE only; the elif chain doesn't
detect that conflict.

### `docker/supervisord.conf:11`

Same elif inserted into the supervisord `[program:main]` command line.
The change is functionally correct but the line is now a 350-char
single-line shell command with three nested `if/elif/else` branches —
that string is going to keep growing every time a new `USE_X` mode is
added. Worth a follow-up to factor it into a small helper script.

### `litellm/__init__.py:138, 154`

```python
from litellm.types.integrations.newrelic import NewRelicInitParams
...
newrelic_params: Optional[Union[NewRelicInitParams, Dict]] = None
```

Module-level config knob, parallel to existing `*_params` entries.
`Union[NewRelicInitParams, Dict]` is consistent with how other
integrations expose typed-or-dict config.

### `litellm/integrations/newrelic/newrelic.py:278–306` — `NewRelicLogger.__init__`

```python
class NewRelicLogger(CustomLogger):
    _last_metric_emission_time: float = 0.0
    _metric_lock = threading.Lock()

    def __init__(self, **kwargs):
        ...
        for k, v in dict_newrelic_params.items():
            kwargs.setdefault(k, v)
        super().__init__(**kwargs)
```

Class-level `_metric_lock` and `_last_metric_emission_time` for the
periodic supportability metric emission. Comment correctly notes that
`setdefault` is used (rather than overwrite) so that explicit
constructor kwargs like `turn_off_message_logging=True` win over the
module-level `litellm.newrelic_params` defaults. Important for the
docstring example in `litellm/__init__.py:243` where the explicit
ctor kwarg is the documented entry point.

### `litellm/integrations/newrelic/newrelic.py:362–371` — content opt-out

```python
return (not self.turn_off_message_logging) and self._parse_bool_env(
    "NEW_RELIC_AI_MONITORING_RECORD_CONTENT_ENABLED", True
)
```

AND-gates two independent off-switches: the constructor kwarg and the
env var, defaulting to record-content=ON. Either disabling the env var
or setting `turn_off_message_logging=True` opts out of content capture.
Read at call time (not init time), which the docstring explicitly
guarantees so that UI config flips take effect without a restart.

### `litellm/integrations/newrelic/newrelic.py:404–451` — supportability metric

Periodic `app.record_custom_metric(metric_name, 1)` emission gated by
both `_last_metric_emission_time` and the `_metric_lock`. The
double-checked-locking pattern at line 447 (`with NewRelicLogger._metric_lock:
... current_time = time.time(); ... - NewRelicLogger._last_metric_emission_time`)
is right — outer non-locked check is the fast path, inner re-check
prevents the metric from being emitted twice if two threads cross the
threshold simultaneously.

### `litellm/litellm_core_utils/custom_logger_registry.py:1116, 1124`

Registers the string key `"newrelic"` → `NewRelicLogger`. Standard
pattern matching how `"datadog"`, `"langfuse"`, etc. are registered.

### `litellm/litellm_core_utils/litellm_logging.py:1136–1160`

Two call sites: one where the registry resolves a callback, and one
that constructs a `NewRelicLogger()` if the existing callback list
doesn't already contain one. The `isinstance(callback, NewRelicLogger)`
dedup check prevents stacking two loggers if a user both lists
`"newrelic"` in `litellm.callbacks` AND sets
`litellm.success_callback = ["newrelic"]`.

## What's good

- Symmetry with the existing `USE_DDTRACE` path is exact — entrypoint
  shell, supervisord command, env-var name shape, module-level
  `*_params`, registry key, and dedup logic in `litellm_logging.py`
  all follow the established APM-integration template. Reviewers can
  audit this PR by diffing against the DDTRACE wiring.
- Two-level content opt-out (kwarg AND env var, both defaulting to
  ON) gives ops teams a runtime kill switch without code changes,
  which is the right ergonomic for compliance scenarios where you
  may need to drop content recording mid-incident.
- Read-at-call-time semantics for `_should_record_content()` are
  documented in the docstring, which prevents the future
  "why did my UI toggle do nothing?" bug.
- Class-level `_metric_lock` + `_last_metric_emission_time` rather
  than instance-level is the right call for a metric that should
  emit at most once per interval *per process*, regardless of how
  many `NewRelicLogger` instances exist.

## Concerns

1. `prod_entrypoint.sh:9–14`: the entrypoint silently prefers
   `USE_DDTRACE` if both flags are set. Would be safer to fail with
   `echo "Error: USE_DDTRACE and USE_NEWRELIC are mutually exclusive"
   >&2; exit 1`. Trying to run a process under both `ddtrace-run` and
   `newrelic-admin` simultaneously would be undefined behaviour
   anyway, and silent precedence will make support tickets harder.

2. `docker/supervisord.conf:11`: the `[program:main] command=` is now
   a ~350-character one-liner with three nested branches and three
   copies of the proxy invocation. Factor into a small helper
   (`docker/run_proxy.sh` that internally does the if/elif/else)
   before this grows a fourth APM mode. Not a blocker for this PR —
   the existing line was already a two-branch one-liner — but it's
   worth a follow-up.

3. `litellm/integrations/newrelic/newrelic.py:347` — `dict_newrelic_params`
   is built from `litellm.newrelic_params` at every `__init__` call.
   If a user instantiates `NewRelicLogger` repeatedly (e.g. test
   fixtures, hot-reload) and `litellm.newrelic_params` is a dict
   that's been mutated in between, the second instance will see the
   mutated version. This is likely intentional and matches how other
   `*_params` work, but worth a docstring note that `litellm.newrelic_params`
   is read-by-reference, not copied.

4. `_metric_lock` is `threading.Lock()`. If the proxy is run under
   asyncio (which it is — uvicorn), and the success/failure callbacks
   land on the main event loop thread, the lock is uncontended in
   practice. Worth a comment explicitly noting "we use threading.Lock
   not asyncio.Lock because callbacks may be invoked from sync worker
   threads as well as the event loop" — otherwise a future contributor
   may "fix" it to `asyncio.Lock` and break the sync callers.

5. The `+2174 / −2` size includes the regenerated `uv.lock` and
   pyproject changes — no concerns there, but it's worth noting the
   actual review surface is closer to ~600 lines of new Python in
   `newrelic.py` plus the integration tests.

## Risk

Low to moderate. The integration is opt-in (both `USE_NEWRELIC=true`
*and* `"newrelic"` in callbacks must be set), so existing deployments
are unaffected. Risk is concentrated in (a) the metric-emission lock
behaviour under load and (b) the `dict_newrelic_params` setdefault
ordering — both of which are testable and the PR claims a test suite
under `tests/test_litellm/integrations/newrelic/`.

## Verdict

**merge-after-nits**

The integration is well-structured and consistent with the existing
APM-integration template. Concerns 1 (mutual-exclusion check in the
entrypoint) and 3 (docstring on `dict_newrelic_params` reference
semantics) are easy fixes that prevent operator footguns. Concern 2
is a follow-up. The lock pattern in §4 is correct as-is, just under-
documented.

## What I learned

The "supportability metric on a class-level lock + last-emission-time"
pattern is a nice way to shed metric-emission load when many logger
instances exist in the same process — better than the more obvious
"emit on every call" or "emit from a background timer" alternatives,
both of which have worse worst-case behaviour under multi-tenant load.
The double-checked locking at the inner `with` block is the bit that
makes it actually safe; the outer non-locked check is just a fast-path
optimization.
