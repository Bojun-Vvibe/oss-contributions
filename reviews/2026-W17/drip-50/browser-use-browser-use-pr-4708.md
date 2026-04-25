---
pr: 4708
repo: browser-use/browser-use
sha: a99fe53bae878a54cccb70dff3ffff2bff87eb79
verdict: merge-as-is
date: 2026-04-25
---

# browser-use/browser-use#4708 — fix: fail fast for unimplemented traces_dir

- **URL**: https://github.com/browser-use/browser-use/pull/4708
- **Author**: cuyua9

## Summary

The `BrowserProfile` Pydantic model exposed a public `traces_dir`
(alias `trace_path`) parameter and even shipped an example using
it (`examples/features/process_agent_output.py`), but no runtime
consumer ever read the value — set it, get a silent no-op. This PR
adds a Pydantic `@model_validator(mode='after')` that raises
`ValueError` if `traces_dir is not None`, deletes the misleading
example line, and adds one regression test.

## Reviewable points

- `browser_use/browser/profile.py:446` — new
  `reject_unimplemented_traces_dir()` validator. Runs `mode='after'`
  so it sees the resolved field; raises with a useful message
  `'traces_dir/trace_path is not implemented yet; remove the
  parameter instead of relying on a silent no-op'`. Note this also
  catches the alias `trace_path` because Pydantic resolves aliases
  to the field before `mode='after'` runs — good, that's the whole
  point.

- `examples/features/process_agent_output.py:24` deletes the
  `traces_dir='./tmp/result_processing',` line. Without this
  deletion the example would now crash, so the change is
  necessary and correct. Worth scanning the rest of `examples/`
  and `docs/` for any other surviving references — a quick
  `grep -r traces_dir docs/ examples/` would catch them. The PR
  shows only the one example file changed; if there are other
  surviving doc references they'll start producing
  `ValueError`s for users following the docs.

- `tests/ci/security/test_security_flags.py:11` adds
  `test_traces_dir_is_rejected_until_implemented` which asserts
  `pytest.raises(ValueError, match='traces_dir/trace_path is not implemented yet')`
  on `BrowserProfile(traces_dir=tempfile.mkdtemp(prefix='test-traces-'))`.
  Good error-message regex match — it's tied to a stable substring,
  not the full string, so future polish to the message won't
  break the test.

- The validator location (under
  `tests/ci/security/test_security_flags.py`) is a minor naming
  smell — this isn't really a security flag, it's a "public-API
  honesty" guard. But the test file already groups validator
  smoke tests under `TestBrowserProfileDisableSecurity`, so it's
  not worth bikeshedding for a 7-line test.

- One small consideration: the validator hard-bans `traces_dir is
  not None`. If a user has a long-running config file that sets
  `traces_dir: null` explicitly to "turn it off", that still
  passes (because `None` is the un-set default). If anyone has
  it set to an empty string (`traces_dir: ""`), the `is not None`
  check catches them and they get the ValueError, which is
  probably what we want — empty-string traces_dir was never going
  to do anything useful either.

## Rationale

Textbook fail-loud-instead-of-silent fix. Three files, atomic,
test-locked, deletes the misleading example that motivated the
silent-bug discovery. The test message-match locks the user-facing
guidance ("remove the parameter") so a future code change can't
silently weaken the message into something like
`'traces_dir not supported'` that'd leave users confused about
whether to remove or keep it.

## What I learned

"Public field with no implementation behind it" is a recurring
trap in Pydantic-modelled config surfaces because the model itself
*looks* like the implementation — the field is typed, it accepts
values, it round-trips through serialization. Users assume "it's
in the schema, therefore it works". The right fix here is exactly
what this PR does: a `@model_validator(mode='after')` raising a
`ValueError` with action-able guidance ("remove the parameter").
The alternative — leaving the field silently ignored and "fixing
it later" — accumulates user configs in the wild that you'll later
have to break loudly anyway, except by then there's a much larger
surface to migrate. Better to break early and explicitly than late
and silently.
