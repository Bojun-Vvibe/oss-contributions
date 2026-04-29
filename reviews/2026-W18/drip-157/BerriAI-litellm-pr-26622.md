# BerriAI/litellm PR #26622 — feat(proxy): add --timeout_worker_healthcheck flag for uvicorn worker triage

- **PR**: https://github.com/BerriAI/litellm/pull/26622
- **Author**: ryan-crabbe-berri
- **Merged**: 2026-04-27T19:13:14Z
- **Head SHA**: `84527b0135df`
- **Size**: +98/-0 across 2 files (`litellm/proxy/proxy_cli.py` + `tests/test_litellm/proxy/test_proxy_cli.py`)
- **Verdict**: `merge-after-nits`

## Context

Operators running `litellm proxy` with `--num_workers > 1` against uvicorn
≥ 0.37.0 want to surface the new `timeout_worker_healthcheck` knob (the
parameter that controls how long the master process waits for a worker
heartbeat before declaring it stuck and recycling it). The parameter
exists upstream in uvicorn but is not exposed through the litellm CLI
unless the proxy adds it explicitly to its uvicorn-args plumbing. The
prior versioned-pin at `setup.py` is `uvicorn>=0.34,<1.0` (looser than
0.37.0), so the flag has to degrade gracefully when an older uvicorn is
installed; otherwise users on the lower bound of the version range get
import-time crashes.

## What changed

1. **CLI option** (`litellm/proxy/proxy_cli.py:583-595`):

   ```python
   @click.option(
       "--timeout_worker_healthcheck",
       default=None,
       type=int,
       help=(
           "Set the uvicorn worker health-check timeout in seconds (...). "
           "Requires uvicorn>=0.37.0. Only applies when running uvicorn directly with --num_workers>1; "
           "ignored under --run_gunicorn / --run_hypercorn."
       ),
       envvar="TIMEOUT_WORKER_HEALTHCHECK",
   )
   ```

   New click option threaded through `run_server(...)` at line 663 and
   forwarded into `_get_default_unvicorn_init_args(...)` at line
   1002-1010, gated on `running_uvicorn = run_gunicorn is False and
   run_hypercorn is False` so the flag is silently dropped under the
   alternative servers (matching the help-text contract).

2. **Capability probe at the helper** (`litellm/proxy/proxy_cli.py:158-167`):

   ```python
   if timeout_worker_healthcheck is not None:
       if (
           "timeout_worker_healthcheck"
           in inspect.signature(uvicorn.Config.__init__).parameters
       ):
           uvicorn_args["timeout_worker_healthcheck"] = timeout_worker_healthcheck
       else:
           print(  # noqa
               f"\033[1;33mLiteLLM Proxy: --timeout_worker_healthcheck "
               f"requires uvicorn>=0.37.0, but installed uvicorn=={uvicorn.__version__}. "
               f"Ignoring the flag.\033[0m"
           )
   ```

   The capability check via `inspect.signature(uvicorn.Config.__init__)`
   is the right shape — version comparison via packaging metadata would
   be more brittle (uvicorn could add the param to a backport, or a
   monkey-patch could remove it) and slower than reading the live
   signature. The graceful-degrade path emits a yellow-ANSI warning to
   stdout and proceeds.

3. **Test coverage** (`tests/test_litellm/proxy/test_proxy_cli.py`):
   adds `test_get_default_unvicorn_init_args` extension at lines
   126-132 (patches `uvicorn.Config` with a fake whose `__init__`
   accepts `timeout_worker_healthcheck`, asserts it ends up in the
   returned args dict) and a new
   `test_timeout_worker_healthcheck_flag` at lines 422-477 that
   round-trips the click flag through `CliRunner` and asserts the
   helper was called with the right kwarg.

## Why `merge-after-nits`

The shape is right (capability-probe via live signature, ANSI-warn-and-
continue on incapable versions, server-mode mutual exclusion handled at
the call site), the help text accurately constrains the contract, and
the test patches `uvicorn.Config` rather than monkey-patching the
inspect call directly — which is the cleaner technique. Three nits:

- **Capability probe runs once per `run_server` invocation; cache it.**
  `inspect.signature(uvicorn.Config.__init__)` is not free (it walks
  the class MRO and parses annotations); for a single-shot CLI it's
  fine, but if `_get_default_unvicorn_init_args` ever gets reused in a
  hot path or unit-test loop, this becomes a cost. Stash the supported
  set as `_UVICORN_SUPPORTED_PARAMS = frozenset(inspect.signature(...).parameters)`
  at module scope.

- **Warning channel is `print` to stdout, not the structured logger.**
  Most other operator-facing warnings in `proxy_cli.py` route through
  `verbose_logger`/`verbose_proxy_logger`. Mixing one yellow-ANSI
  `print()` into an otherwise structured-log surface means the warning
  won't show up in JSON-log shipping pipelines. Either drop the ANSI
  and use `verbose_proxy_logger.warning(...)`, or document explicitly
  that this is "user-facing CLI banner, not log signal" and add a
  matching `--no-color` respect.

- **`running_uvicorn` derivation reads slightly off-script.** At
  `proxy_cli.py:1002`:

  ```python
  running_uvicorn = run_gunicorn is False and run_hypercorn is False
  ```

  These are typed booleans elsewhere in the file but the explicit
  `is False` rather than `not run_gunicorn` is defensive against
  `None`/`""`/`0` truthiness. That's correct, but worth a one-line
  comment because the next reader will instinctively "simplify" it
  to `not run_gunicorn and not run_hypercorn` — silently broken if
  click ever passes a `None` default.

## What I learned

When a CLI flag forwards a kwarg into a downstream library that may or
may not support it (depending on installed version), the runtime-shape
options are: (1) version-pin the dep tighter, (2) version-string parse
at runtime, or (3) capability-probe via `inspect.signature`. Option (3)
is the most robust — it survives backports, monkey-patches, and
beta-channel installs that don't bump the version string — and it's
trivially testable because you patch the target class with a fake whose
`__init__` has the signature you want to test against. The pattern is
worth stealing whenever you find yourself thinking "I want to use a
feature that's only in newer versions of $LIB."
