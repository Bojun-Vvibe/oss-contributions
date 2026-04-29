# BerriAI/litellm #26776 — harden `safe_deep_copy` against concurrent mutation + single-instance `_ProxyDBLogger`

- **PR:** https://github.com/BerriAI/litellm/pull/26776
- **Title:** `fix(proxy): harden safe_deep_copy against concurrent mutation + single-instance _ProxyDBLogger`
- **Author:** bse-ai (Brendan Smith-Elion)
- **Head SHA:** 630df3b73a8221961236c8ce9a639fbf529e4eef
- **Files changed:** 5 (`litellm/litellm_core_utils/core_helpers.py` +66/−36, `litellm/proxy/guardrails/guardrail_hooks/noma/noma_v2.py` +20/−5, `litellm/proxy/proxy_server.py` +9/−4, two test files +182/0), **+277 / −45**
- **Verdict:** `merge-after-nits`
- **Re-files:** #26215 (closed for unsigned CLA — same code rebased)

## What it does

Three independent proxy fixes bundled because they share test scaffolding
and were all uncovered tracing one downstream spend-attribution incident.

### Fix 1 — Snapshot-before-iterate in `safe_deep_copy` (`core_helpers.py:288-372`)

Before: `safe_deep_copy(data)` iterated `data.items()` directly and
pop/re-inserted `litellm_parent_otel_span` on `data["metadata"]` and
`data["litellm_metadata"]` *in place*. Two bugs in one:

1. Concurrent hooks/callbacks mutating those dicts during the iteration
   raised `RuntimeError: dictionary changed size during iteration`.
   Author cites ~40/day in production at a downstream deployment.
2. The pop/re-add dance mutated the **caller's** nested dict — the
   placeholder swap was visible to anyone else holding a ref to
   `request_data["metadata"]`.

After: `dict(data)` snapshot at line 313 (atomic under the GIL), then
`dict(metadata)` and `dict(litellm_metadata)` snapshots at lines 316-321
of the two nested dicts that are known-mutable. All subsequent pop /
re-insert / iterate operations work on the snapshot, never the caller's
data. Top-level non-dict inputs short-circuit at lines 309-313 to a plain
`copy.deepcopy` with the same fallback-to-ref behavior.

The new `test_build_scan_payload_snapshots_request_data_and_metadata`
(`test_noma_v2.py:240-275`) pins the in-place-mutation regression with a
sentinel object and asserts `metadata is request_data["metadata"]`
remains true post-build. The `test_build_scan_payload_concurrent_mutation_does_not_raise`
threading repro (lines 277-330) reliably reproduces the
`RuntimeError` on pre-fix code within a few hundred iterations.

### Fix 2 — Single shared `_ProxyDBLogger` (`proxy_server.py:1762-1776`)

Before: two separate `_ProxyDBLogger()` instances were registered, one
on `litellm.callbacks` and one on `litellm._async_success_callback`.
Functionally fine (each list iterates independently so writes weren't
doubled) but `/active/callbacks` showed the hook at two distinct object
addresses, which actively blocked triage of the spend-tracking incident.
After: one `logger = _ProxyDBLogger()` is bound to a local and registered
on both lists.

### Fix 3 — `verbose_proxy_logger.exception` in Noma v2 (`noma_v2.py:338`)

`verbose_proxy_logger.error("Noma v2 guardrail failed: %s", str(e))`
was swallowing tracebacks, making it impossible to distinguish "Noma API
returned 5xx" from "local concurrent-mutation race during payload
build" — exactly the case the author was trying to triage. Switched to
`.exception(...)` for traceback capture.

## Why it's correct

- The snapshot pattern in Fix 1 is the textbook fix for "iterate a dict
  that other threads can mutate" under CPython's GIL: `dict(d)` is
  atomic and produces a stable view. The author's concurrent-mutation
  test is a real, reliable, mechanical reproducer (timing-bounded but
  the failure mode is deterministic on pre-fix code).
- Fix 2's behavior change is purely cosmetic at the I/O level (writes
  were already not doubled) but materially improves observability — and
  is the kind of "we couldn't tell what we were looking at" finding that
  belongs in the same PR as the underlying bug.
- Fix 3 is one method-name swap with an obvious and correct rationale.
- Re-filing on a fresh branch with a signed CLA is the right way to
  recover from the #26215 bulk-close. Greptile-passed prior art is
  noted.

## What's good

- The 9-line module-doc rewrite at `core_helpers.py:288-302` explains
  the *why* (atomicity under the GIL, two known-mutable nested dicts)
  not just the *what*. Future readers will not need to dig through git
  blame.
- Inline comments at `core_helpers.py:323-327, 334-339, 351-354,
  363-366` walk through Step 0 / 1 / 2 / 3 in order, each explaining
  what is and isn't being mutated.
- The pessimism in the Step 2 comment ("Nested dicts beyond the two we
  snapshotted are still copied via `copy.deepcopy`, which iterates
  them; if those are also concurrently mutated this can still raise.
  Known callers only mutate metadata/litellm_metadata concurrently, so
  we cover the realistic surface without paying for a full recursive
  snapshot.") is the right level of honesty about the partial fix.
- The Noma v2 caller (`noma_v2.py:142-156`) is updated to also snapshot
  `model_call_details` before assigning into `payload_request_data` and
  switches to a non-mutating spread-construct pattern (`{**payload, "litellm_logging_obj": ...}`)
  instead of `payload[k] = v` — fixes the symmetric in-place-mutation
  bug at the call site.

## Nits / risks

1. **Bundle the three fixes is borderline.** Fix 1 + Fix 3 share the
   incident root cause and belong together. Fix 2 (single
   `_ProxyDBLogger`) is genuinely independent — only the *triage* of
   the incident motivated it, not the bug itself. A maintainer who
   wants to revert Fix 2 because of an unrelated callback-ordering
   regression would have to revert the whole PR or do a partial revert.
   Splitting Fix 2 into a follow-up PR is the cleaner path; if it's
   bundled, add a CHANGELOG line so the cosmetic-but-observable change
   is discoverable.
2. **`test_build_scan_payload_concurrent_mutation_does_not_raise` is
   timing-bounded** (`deadline = time.time() + 0.5`) and the PR
   acknowledges it ("flaky by construction — it times out rather than
   failing"). The repro relies on the pre-fix bug being triggered
   within 500ms. On a slow CI runner under load, the test may pass
   even on the pre-fix code (false negative). Consider either
   (a) marking it `@pytest.mark.timing_sensitive` and excluding from
   the default suite, or (b) using `pytest-repeat` to run 10x and
   asserting the pre-fix path raised at least once during a manual
   bisect (not in CI).
3. **Step 2 partial-snapshot risk is real.** The honest comment at
   `core_helpers.py:344-348` admits that nested dicts other than
   `metadata`/`litellm_metadata` can still race. A follow-up should add
   a `safe_deep_copy_recursive_snapshot()` variant for callers that
   need the full guarantee, and a unit test that exhaustively walks
   `litellm.proxy` for concurrent-write sites on dicts other than the
   two whitelisted ones. Worth at least filing the issue.
4. **`isinstance(data, dict)` short-circuit at `core_helpers.py:309-313`
   masks an existing behavior:** the old code ran the otel-pop dance
   only when `data` was a dict, then fell through to a top-level
   `copy.deepcopy(data)` for non-dicts. New code returns early on
   non-dicts via `try/except` swallow. Confirm no caller passes a
   `pydantic.BaseModel` or `dataclass` instance and expects per-field
   deepcopy fallback — if any do, they now get the all-or-nothing path.
5. **Test-name nit.** `test_build_scan_payload_snapshots_request_data_and_metadata`
   is descriptive but extremely long. Consider
   `test_payload_build_does_not_mutate_caller_metadata`.
6. **PR body is great** (probably the best-written PR body in this
   drip). Keep doing this.

## Verdict

`merge-after-nits` — high-quality bug fix with good tests, honest
inline documentation, and clear motivation. Address the
single-`_ProxyDBLogger` bundling question (split or CHANGELOG-note
it), tag the timing-sensitive test, and file the recursive-snapshot
follow-up as an issue, then ship.
