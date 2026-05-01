# BerriAI/litellm #27012 — fix(guardrails): post-call guardrail must only fire once

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/27012
- **HEAD SHA:** `a1fd150b989f5931d2895c210087ed03780798a7`
- **Author:** ryan-crabbe-berri (rebrand of #26109 from
  mubashir1osmani, brought into a project-owned branch for the
  staging flow)
- **Verdict:** `merge-after-nits`

## What the diff does

Closes a duplicate-fire bug in the deferred-stream-guardrails path
where guardrails that implement `async_post_call_streaming_iterator
_hook` were being run *twice* per request: once during the
streaming pipeline (where the iterator hook runs natively against
each chunk and on completion against the assembled response), and
*again* when `_run_deferred_stream_guardrails` walked the callback
list at end-of-stream and called `cb.async_post_call_success_hook(
...)` on every guardrail without checking whether it had already
been invoked via the iterator path.

The duplicate fire had two real-world consequences:
- Guardrails that hit external paid moderation APIs (OpenAI
  Moderation, third-party safety services) were billed twice per
  request.
- Guardrails that *passed* on the streaming-iterator pass could
  spuriously *fail* on the deferred-success pass (different
  response framing, race against partial-buffer state, etc.),
  blocking responses the streaming check had already approved.

Fix: at `litellm/proxy/common_request_processing.py:1604-1615`
(inside the `_run_deferred_stream_guardrails` callback loop), add
a second `continue` arm right after the existing
`async_pre_call_hook` skip check:

```python
if "async_post_call_streaming_iterator_hook" in type(cb).__dict__:
    # Skip — the guardrail already scanned the assembled
    # response via its own streaming iterator hook in the
    # streaming pipeline. re running this function async_post_call_success_hook
    # here would duplicate the scan and can spuriously block the guardrail that already passed / failed.
    continue
```

The `type(cb).__dict__` check (rather than `hasattr(cb, ...)`) is
the load-bearing detail — `hasattr` would also return True for
methods inherited from a base class, so a guardrail subclass that
doesn't override the iterator hook but inherits an inherited
no-op would also get incorrectly skipped. `type(cb).__dict__`
checks for the method *defined directly on the concrete class*,
which is the correct membership test for "this guardrail
specifically implements iterator-hook-based scanning."

Files of note:

- `litellm/proxy/common_request_processing.py:1604-1615` (+6 /
  -0) — the new skip arm.
- `tests/test_litellm/proxy/guardrails/test_deferred_guardrail_
  logging.py:761-814` (+54 / -0) — new test
  `test_streaming_iterator_hook_skipped_in_deferred_path` that
  constructs an `IteratorHookGuardrail` subclass with both
  `async_post_call_streaming_iterator_hook` (no-op pass-through
  yielding chunks unchanged) and `async_post_call_success_hook`
  (sets a `success_hook_called = True` nonlocal). Calls
  `ProxyBaseLLMRequestProcessing._run_deferred_stream_guardrails`
  with the guardrail patched into `litellm.callbacks`, awaits a
  zero-duration sleep to drain pending tasks, then asserts
  `success_hook_called is False`. Negative-property test (the
  load-bearing one for "must NOT fire twice").

## Why it's right

The diagnosis matches the original symptom (OpenAI Moderation
charges doubling, guardrails spuriously failing in the deferred
path on responses they had passed in the streaming path), and the
fix is at the right call-graph position — the `_run_deferred_
stream_guardrails` loop is the point where the *deferred* execution
ranges over the callback list, and skipping iterator-hook-
implementing guardrails at this point is the minimum-scope way to
say "guardrails that own their own scanning via iterator-hook
shouldn't *also* be invoked via the success-hook path."

The `type(cb).__dict__` membership check is dispositive on the
"is this method defined directly on this concrete class" question
and avoids the subtle `hasattr` over-fire on inherited base-class
methods. A guardrail that wants to inherit from a mixin providing
the iterator hook gets the skip behavior correctly; a guardrail
that doesn't implement iterator-hook-based scanning runs through
the success-hook path as before.

The test is the right shape: it constructs a guardrail with both
hooks defined on the concrete class, runs the deferred path, and
asserts the success hook was NOT called. The `await asyncio.sleep
(0)` at `:807` drains any pending tasks (the original duplicate-
fire happened on the next event-loop tick after the iteration
completed), so the assertion runs after any deferred coroutines
would have scheduled. The mocked `assembled_response = MagicMock()`
is fine because the test is asserting the iterator-skip behavior,
not the response-content semantics.

The comment in the new arm is factually correct ("the iterator hook
already scanned the assembled response in the streaming pipeline,
re running this function async_post_call_success_hook here would
duplicate the scan and can spuriously block the guardrail that
already passed / failed") and names both consequences (duplicate
billing, spurious blocking).

The PR provenance — "Brings @mubashir1osmani's fix from #26109
(closed in favor of this PR) into a project-owned branch so we can
ship it through the normal staging flow" — is a project-internal
process detail; co-author attribution preserved via `Co-authored-by:
mubashir1osmani <mubashir1osmani@users.noreply.github.com>` at the
end of the PR body. That's the right way to handle a fork-author-
to-project-owner handoff.

## Nits

1. **`type(cb).__dict__` is method-membership-on-direct-class
   only.** This is correct for the immediate fix, but if a
   guardrail uses a mixin pattern (`class MyGuardrail(IteratorHook
   ScanningMixin, BaseGuardrail):` where the iterator hook is
   defined on the mixin), `type(cb).__dict__` would NOT see the
   inherited method and would incorrectly run the success hook
   path, duplicating the scan. A more robust check would walk
   the MRO: `any("async_post_call_streaming_iterator_hook" in
   klass.__dict__ for klass in type(cb).__mro__ if klass is not
   object)`. Worth at least a comment explaining the
   direct-class-only constraint and naming the mixin failure mode
   so future guardrail authors know to override the iterator hook
   directly on their concrete class.

2. **Comment grammar** at `:1607-1612` — "re running this function
   async_post_call_success_hook here" is awkward. Cleaner: "Re-
   running `async_post_call_success_hook` here would duplicate the
   scan and can spuriously block a guardrail that already passed
   or failed in the streaming path." Mechanical edit.

3. **Test asserts the negative property only** — good for the
   immediate regression-pin, but a parallel positive-property
   test (`test_success_hook_runs_when_iterator_hook_not_defined`)
   would lock that the existing non-iterator-guardrail path still
   fires on the deferred success-hook. Without the positive lock,
   a future overly-aggressive change to the skip predicate could
   over-skip and silently break legitimate guardrails.

4. **`assembled_response = MagicMock()` at `:801`** — fine for
   this test's purpose (asserting the iterator hook prevents
   `async_post_call_success_hook` from running), but if a future
   change to `_run_deferred_stream_guardrails` starts inspecting
   the response shape *before* the callback loop, this mock could
   silently start passing for the wrong reason. A more realistic
   `assembled_response = ModelResponse(...)` fixture would be
   more robust against future call-graph drift.

5. **`mock_logging_obj.async_success_handler = track_async_success`**
   at `:797` — the no-op coroutine is fine, but its presence
   suggests there's *another* duplicate-call path through the
   logging handler that this test doesn't cover. Worth a
   followup investigation: are there *other* callsites where an
   iterator-hook guardrail's success-hook is being called? grep
   for `async_post_call_success_hook` callers and confirm this
   PR closes them all.

6. **Two co-authors on a project-internal rebrand** is fine, but
   the original PR #26109's review history (any maintainer
   feedback that prompted the rebrand) should be referenced in
   the PR body or commit message so the security-review trail
   stays continuous. A line like "Original PR review feedback
   addressed in commit X" would close that loop.

## Verdict rationale

Right diagnosis (deferred-success-hook re-running iterator-hook-
based guardrails causes double API billing and spurious blocks),
right call-graph position (skip arm in `_run_deferred_stream_
guardrails` callback loop), right primitive (`type(cb).__dict__`
direct-class membership check rather than `hasattr` MRO over-fire),
right test surface (negative property locked with realistic
guardrail subclass, drained event loop, single assertion).
Wants MRO-walking variant to handle mixin-based guardrail patterns
(or explicit comment naming the constraint), comment grammar
cleanup, parallel positive-property test for the non-iterator path,
realistic `assembled_response` fixture, audit of other callsites
where iterator-hook guardrails could be re-fired, and a reference
to original PR #26109's review history for the security trail.

`merge-after-nits`
