# BerriAI/litellm #27018 — [Fix] Isolate dual OTEL handlers

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/27018
- **HEAD SHA:** `8eb1764e7e35620f1b868d5c2dd247ebb7146e2a`
- **Author:** Michael-RZ-Berri
- **Verdict:** `merge-after-nits`

## What the diff does

Closes a dual-handler coexistence bug where two `OpenTelemetry`
callback handlers configured in the same process were silently
ending up wired to the *same* `TracerProvider` because OpenTelemetry
SDK's global-provider singleton swallows whichever provider is
created second. Concretely: handler A would set the global, handler
B would call `_get_or_create_provider` and see "an SDK provider
already exists, reuse it," and any spans handler B emitted would
flow to handler A's exporter instead of its own. Fix: a new
`OpenTelemetryConfig.skip_set_global: bool = False` flag (defaults
False so existing callers get the prior behavior) that, when True,
tells `_get_or_create_provider` to *create a private provider*
instead of reusing the global SDK one.

Files of note:

- `litellm/integrations/opentelemetry.py:69-72` — adds `skip_set_
  global: bool = False` to `OpenTelemetryConfig` with a docstring
  comment ("When True, create a private TracerProvider instead of
  reusing or setting the global one") naming the intent.
- `litellm/integrations/opentelemetry.py:259-285` — the load-
  bearing branch in `_get_or_create_provider`. Prior behavior:
  if the existing global provider is an SDK provider (positive
  type check, not the proxy-no-op type), reuse it unconditionally.
  New behavior: same type check, but if `skip_set_global` is True,
  fall through to `provider = create_new_provider_fn()` to get a
  private one. The else-branch (existing SDK provider, no
  skip_set_global) keeps the prior reuse semantics.
- `litellm/integrations/opentelemetry.py:312-315` — `skip_global`
  derivation now ORs the new flag with the prior callback-name-
  based heuristic (`self.callback_name == "langfuse_otel"`), so
  langfuse-back-compat continues to work without callers needing
  to set `skip_set_global=True` explicitly. The comment updates
  from "CRITICAL FIX: For Langfuse OTEL, skip setting global
  provider to prevent interference" to "langfuse_otel relies on
  the Langfuse SDK's TracerProvider; don't overwrite it" — the
  new comment is more precise about *why* langfuse needs this.
- `litellm/integrations/opentelemetry.py:330` — adds `self._
  tracer_provider = tracer_provider` so the handler holds a
  reference to its own provider. Load-bearing for the test
  surface (assertions like `handler_a._tracer_provider is not
  handler_b._tracer_provider`) and also for runtime correctness:
  without this field, a caller that wants to do
  `force_flush(timeout)` on the specific handler's provider has
  to chase it through the global SDK, which defeats the
  isolation.
- `tests/test_litellm/integrations/test_opentelemetry.py:262-366`
  — new `TestOpenTelemetryDualHandlerIsolation` class with four
  tests:
  - `test_skip_set_global_creates_isolated_provider_when_sdk_
    provider_global` — sets a TracerProvider as the global,
    creates an OpenTelemetry handler with `skip_set_global=True`,
    asserts `handler._tracer_provider is not first_provider`,
    asserts `trace.get_tracer_provider() is first_provider`
    (the global wasn't overwritten), then emits a span through
    the handler's tracer and asserts it shows up in the handler's
    own exporter (not the global one).
  - `test_skip_set_global_via_callback_name_back_compat` — the
    langfuse-name-based heuristic still works without explicitly
    setting the new flag.
  - `test_default_behavior_reuses_existing_sdk_provider` —
    locks the *negative* property: with `skip_set_global=False`
    (default), an existing SDK provider is reused (the prior
    behavior). Critical to have this test because without it, a
    future change that inverts the default would silently break
    callers relying on global-provider sharing.
  - `test_two_handlers_each_receive_their_own_spans` — the
    end-to-end canonical case: handler A (skip_set_global=False)
    + handler B (skip_set_global=True), each gets its own
    exporter, each emits one span, assert each exporter sees
    exactly its own handler's span and nothing else.
- `tearDown`/`setUp` at `:267-275` reset `trace._TRACER_PROVIDER_
  SET_ONCE._done = False` and `trace._TRACER_PROVIDER = None` to
  exercise the env-driven init path on every test (otherwise the
  set-once gating would short-circuit).
- `_wire_handler_exporter` helper at `:278-296` is a context
  manager that monkey-patches `OpenTelemetry._get_span_processor`
  to return a `SimpleSpanProcessor(exporter)` for the duration of
  one handler's construction. This is the right test primitive
  for "give this handler a known exporter without going through
  env-var or constructor wiring."

## Why it's right

The diagnosis is correct: OpenTelemetry SDK's global provider is
a process-wide singleton (set via `trace.set_tracer_provider()`,
guarded by `_TRACER_PROVIDER_SET_ONCE`), and the prior reuse-an-
existing-SDK-provider branch made dual-handler coexistence
impossible — two LiteLLM handlers wanting separate exporters would
silently end up sharing one provider, with all of handler B's
spans going to handler A's exporter (or vice versa, depending on
construction order).

The fix shape is right: `skip_set_global` is opt-in (`= False`
default) so existing callers see no behavior change, and when set
True, the load-bearing branch at `:264-274` calls
`create_new_provider_fn()` to get a private provider rather than
reusing the global. The langfuse back-compat OR (`self.config.
skip_set_global or self.callback_name == "langfuse_otel"`) at
`:312-314` keeps the existing langfuse special-case working
without callers needing to update their config — the langfuse SDK
sets its own provider as the global and LiteLLM's existing
behavior was already to skip global-set when the callback name
matched.

The `self._tracer_provider = tracer_provider` assignment at `:330`
is the load-bearing handle for the isolation property: without it,
a caller doing per-handler `force_flush()` or holding a reference
to "my provider" has nowhere to look except the global SDK, which
in the dual-handler case wouldn't even be the right one. The test
suite uses this field heavily (`handler_a._tracer_provider is not
handler_b._tracer_provider`).

The four-test surface is well-chosen: the *positive* property
(skip_set_global creates an isolated provider) gets two tests
covering both code paths to it (explicit flag + langfuse-name
back-compat), the *negative* property (default behavior reuses the
global) gets its own test (load-bearing — without this, future
default-flip would silently regress shared-global users), and the
*end-to-end* property (two handlers, two spans, two exporters,
each gets exactly its own) is the canonical "this is what the bug
report described" regression-pin.

The `_wire_handler_exporter` monkey-patch helper is the right test
primitive for this layer — exporting via the env-var path would
require setup/teardown of OS-level env state and would make the
tests flaky in parallel runs; constructor-time injection of the
exporter is what the test wants but the production constructor
doesn't expose that surface; monkey-patching `_get_span_processor`
for one construction is the minimum-diff way to pin the exporter
without changing the production API.

The comment update at `:312-313` ("langfuse_otel relies on the
Langfuse SDK's TracerProvider; don't overwrite it") is more
informative than the prior "CRITICAL FIX: ... to prevent
interference" — the new wording names *why* (the Langfuse SDK
already set its own provider as the global and we don't want to
displace it) rather than vague "interference."

## Nits

1. **`skip_set_global` flag name** — reads as "skip setting the
   global" but the actual semantic is two-fold: (a) don't set our
   provider as the global, *and* (b) don't reuse an existing
   global SDK provider, instead create a private one. The "create
   a private provider when the global already exists" half of the
   semantic is the surprising bit and isn't in the flag name. A
   name like `isolated_provider` or `private_provider` would
   better express the user-facing intent. (Migration: keep
   `skip_set_global` as a deprecated alias if shipped.)

2. **`tearDown`/`setUp` reach into OpenTelemetry internals**
   (`trace._TRACER_PROVIDER_SET_ONCE._done = False`,
   `trace._TRACER_PROVIDER = None`) — necessary given the global-
   singleton design but fragile against OpenTelemetry SDK upgrades
   (these are private attrs and could be renamed). A wrapper
   helper `_reset_otel_global_state()` with a comment naming the
   OpenTelemetry version this was written against would localize
   the breakage when the SDK changes shape.

3. **No test for the `force_flush` per-handler use case** that
   the `_tracer_provider` field exists to support. The four
   tests assert each handler's spans go to the correct exporter,
   but a fifth test pinning "calling `handler_a._tracer_provider.
   force_flush()` only flushes A's spans, not B's" would lock the
   load-bearing isolation property at the runtime-control surface
   that operators actually use during shutdown.

4. **`isinstance(existing_provider, sdk_provider_class)` check at
   `:264`** — this branch handles "an SDK-class provider exists
   globally." The else-branch ("default proxy provider or unknown
   type, create our own") at `:280-285` doesn't gate on
   `skip_set_global` because it always creates a new one anyway.
   That's correct but a brief comment ("`skip_set_global` only
   matters when there's already an SDK provider to reuse — the
   no-existing-provider path always creates a new one regardless")
   would prevent a future "shouldn't this branch also check
   skip_set_global?" question.

5. **The `langfuse_otel` callback-name string check at `:313`**
   is an implicit-coupling antipattern: a typo or rename would
   silently disable the back-compat. Worth introducing a typed
   `LangfuseOtelCallbackName` constant exported from the langfuse
   integration module, so the string lives in one place and a
   refactor would have a compile-time-ish find-references handle.

6. **No docs update** for the new `skip_set_global` config field
   in the LiteLLM observability docs — operators reading the
   OpenTelemetryConfig docs page won't discover this option. PR
   should include a docs snippet showing the dual-handler
   coexistence pattern.

## Verdict rationale

Right diagnosis (OpenTelemetry SDK global-singleton swallowing the
second handler's provider in dual-handler configs), right shape
(opt-in flag preserving prior behavior on default, load-bearing
branch in `_get_or_create_provider`, langfuse back-compat
preserved via OR with the existing callback-name heuristic),
right load-bearing test surface (positive case via two paths,
negative case for the prior-behavior lock, end-to-end two-handler
two-exporter assertion), `_tracer_provider` field added for
runtime control surface needs. Wants flag-name better aligned
with the "create-private-not-reuse-global" semantic, OpenTelemetry-
internal-state reset wrapped in a versioned helper, force_flush
isolation test, comment on why the no-existing-provider branch
doesn't need the flag check, typed constant for the langfuse
back-compat string, and a docs update.

`merge-after-nits`
