---
pr: BerriAI/litellm#26741
sha: 3215874e400de2989db7b15ef85c27dd703381af
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(test): scope ERROR log assertion to LiteLLM logger in test_model_alias_map

URL: https://github.com/BerriAI/litellm/pull/26741
Files: `tests/local_testing/test_model_alias_map.py`
Diff: 3+/4-

## Context

`test_model_alias_map` was asserting that no record in `caplog` had
`levelname == "ERROR"` across the entire process. Pytest's `caplog`
fixture captures every logger emitting via the root handler, so any
unrelated background task in the same test session — most often
`asyncio` records like `Unclosed client session` / `Unclosed connector`
emitted from teardown of HTTP clients in sibling tests, or third-party
observability integrations — would trip the assertion. Those records
have `rec.name == "asyncio"`, not `"LiteLLM"`, so the failures were
both flaky and a false positive against the code under test.

## What's good

- The new predicate at `test_model_alias_map.py:33-35` correctly
  narrows to `rec.name.startswith("LiteLLM")`, which is the namespace
  the LiteLLM library uses for its loggers (`LiteLLM`, `LiteLLM Proxy`,
  `LiteLLM Router`). Any LiteLLM-emitted ERROR will still flunk the
  test, preserving the test's original intent.
- `pytest.fail(f"Unexpected litellm ERROR log: {rec.getMessage()}")`
  at `:35` is a strict improvement on the prior `assert "ERROR" not
  in log` — failures now name the offending message instead of just
  the level, which is exactly the diagnostic any future maintainer
  needs to triage.
- The replacement loop iterates `caplog.records` directly instead of
  pre-projecting to `[rec.levelname for rec in caplog.records]`,
  which keeps both `levelname` and `name` available without a second
  pass.

## Nits / follow-ups

- The PR body correctly notes that the underlying `Unclosed
  ClientSession` warnings are real bugs worth fixing separately —
  agreed. They indicate that some LiteLLM code path or test helper
  is not awaiting `client.close()` / `await session.close()` on
  shutdown, and silently leaking those at process exit is a
  resource-cleanup bug that just happens to surface here as test
  flakiness. Worth tracking as a follow-up issue.
- `rec.name.startswith("LiteLLM")` does the right thing today; if
  the library ever standardizes on `litellm.*` (lowercase) for new
  loggers (PEP 8 module-name convention), this predicate will silently
  stop catching them. A `("LiteLLM", "litellm")` tuple to
  `startswith()` would be future-proof.

## Verdict

`merge-as-is` — single-file 7-line targeted test fix that addresses
a real flake source without coupling to unrelated async-cleanup
noise, with a strictly better failure message. Exactly the right
shape for a deflake PR.
