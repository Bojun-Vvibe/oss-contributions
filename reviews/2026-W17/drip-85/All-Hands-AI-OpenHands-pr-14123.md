# All-Hands-AI/OpenHands PR #14123 — feat: Auto-forward LMNR_* environment variables to agent-server

- **Repo:** All-Hands-AI/OpenHands
- **PR:** [#14123](https://github.com/All-Hands-AI/OpenHands/pull/14123)
- **Files:** 2 files, +73/−4
  - `openhands/app_server/sandbox/sandbox_spec_service.py` (+8/−2)
  - `tests/unit/app_server/test_agent_server_env_override.py` (+62/−0 in a new test class, +3 unrelated touch)

## Context

OpenHands' V1 two-container architecture (app-server + agent-server)
auto-forwards a small allow-list of env-var prefixes from app-server's
process env into the agent-server container. Previously only `LLM_*`
was on that list. This PR adds `LMNR_*` (Laminar — an open-source
LLM monitoring/analytics service) so users running the V1 split
architecture can configure Laminar via env without writing a JSON
override into `OH_AGENT_SERVER_ENV`.

## What the diff actually does

**`sandbox_spec_service.py:74`** — single-line tuple extension:
```diff
-AUTO_FORWARD_PREFIXES = ('LLM_',)
+AUTO_FORWARD_PREFIXES = ('LLM_', 'LMNR_')
```

**Doc-comment update at `:80-101`** adds a paragraph explaining
that LMNR_* covers Laminar monitoring/analytics, lists
`LMNR_PROJECT_API_KEY` and `LMNR_BASE_URL` as canonical examples,
and adds the prefix to the bullet-list at `:91-94`.

**Test class `TestLMNRAutoForwarding` at
`test_agent_server_env_override.py:298-356`** — five new tests:
1. `test_auto_forward_prefixes_contains_lmnr` — direct assertion
   on the constant.
2. `test_lmnr_project_api_key_auto_forwarded` — fixture
   `{'LMNR_PROJECT_API_KEY': 'sk-test-key-12345', 'OTHER_VAR': 'should_not_be_included'}`,
   asserts the LMNR var is forwarded and OTHER_VAR is not.
3. `test_lmnr_base_url_auto_forwarded` — same shape for
   `LMNR_BASE_URL`.
4. `test_multiple_lmnr_vars_auto_forwarded` — three vars at once.
5. `test_lmnr_prefix_is_case_sensitive` — fixture includes
   `LMNR_PROJECT_API_KEY` (uppercase), `lmnr_project_api_key`
   (lowercase), `Lmnr_Project_Api_Key` (mixed case); only the
   uppercase variant is forwarded.

All tests use `with patch.dict(os.environ, env_vars, clear=True):`
and call `get_agent_server_env()`.

## Strengths

- **Minimum-surface change.** A one-line tuple extension at
  `:74` plus proportional doc/test additions. No new code path,
  no new branches, no abstraction churn.
- **Test parity with existing `LLM_` prefix tests** — the new
  `TestLMNRAutoForwarding` class mirrors the structure of
  `TestLLMAutoForwarding` (probably exists earlier in the file)
  including the deliberate case-sensitivity assertion. This keeps
  the contract between the two prefixes symmetric and locks the
  case-sensitive guarantee against future "should be
  case-insensitive" feature requests that might silently leak
  third-party env vars.
- **Doc comment is integrated, not appended** — `:80-94` updates
  in place rather than tacking on a "Note: also LMNR_" footnote,
  so the file reads as a coherent description of the
  auto-forwarding contract.
- **`patch.dict(os.environ, env_vars, clear=True)`** at every
  test means each test starts from a clean env, so test order
  doesn't matter and parallel-test-runners (pytest-xdist) won't
  see cross-pollution.

## Risks / nits

1. **Allow-list approach scales linearly with monitoring vendors.**
   Today it's `LLM_` and `LMNR_`; tomorrow it'll be `OTEL_`,
   `LANGFUSE_`, `LANGSMITH_`, `HELICONE_`, `BRAINTRUST_`, etc.
   Each one needs an unrelated PR with the same one-line tuple
   change plus 60 lines of test-class boilerplate. Worth a
   follow-up issue: should the allow-list become a configurable
   env var itself (`OH_AUTO_FORWARD_PREFIXES`), so users can opt
   into vendor prefixes without a code change?
2. **Security implication of forwarding `LMNR_PROJECT_API_KEY`** —
   the agent-server runs untrusted user code (that's the whole
   sandbox model). Forwarding an API key into that container
   means a prompt-injection attack could read it via
   `os.environ`. The `LLM_*` precedent already has this issue
   (LLM API keys are forwarded), so this isn't a new attack
   surface, but the doc-comment should explicitly note the trust
   model: "auto-forwarded prefixes are visible to agent code; do
   not put secrets here that the agent should not see."
3. **`os.environ` is patched but `get_agent_server_env`'s caching
   behavior isn't tested.** If `get_agent_server_env` reads
   `os.environ` lazily, the tests pass. If it caches the env at
   module-import time, the tests would also pass under
   `clear=True` because pytest re-imports the module per test.
   But a real prod scenario where the user sets `LMNR_*` after
   process start would silently not work. Worth one
   `test_lmnr_var_set_after_initial_call_is_picked_up` test that
   calls `get_agent_server_env()` twice with different env state
   between calls.
4. **No `OH_AGENT_SERVER_ENV` precedence test for LMNR_** — the
   doc comment at `:104-105` mentions the JSON-override mechanism
   coexists with auto-forwarding, but there's no test asserting
   "if both `LMNR_BASE_URL` env *and*
   `OH_AGENT_SERVER_ENV='{"LMNR_BASE_URL":"override"}'` are set,
   the JSON override wins". That precedence is presumably tested
   for `LLM_*` already, but this PR is the natural place to lock
   it for `LMNR_*` too.
5. **Test uses string-literal `'sk-test-key-12345'`** as a Laminar
   project key. Even though it's clearly a test fixture, secret
   scanners (truffleHog, gitleaks, GitHub's secret scanning) may
   flag the `sk-` prefix as an OpenAI-style key shape. Use a
   format that won't trip scanners, e.g.
   `'lmnr-test-key-not-real'` or `'fake-key-do-not-use'`.
6. **The `LMNR_VERBOSITY` value `'debug'`** in
   `test_multiple_lmnr_vars_auto_forwarded` is fictional —
   Laminar's actual config doesn't use `LMNR_VERBOSITY` (it uses
   `LMNR_DEBUG=1` or similar). Tests don't strictly need to use
   real env-var names since the test only validates the prefix
   match, but using real names makes the tests serve as
   documentation. Cross-check the Laminar SDK docs or use
   `LMNR_PROJECT_NAME` instead.
7. **No update to `pyproject.toml`** — if Laminar SDK is now an
   intentional integration point, should it be added as an
   optional dependency? Probably not (the env vars are read by
   user code in the agent-server, not by OpenHands itself), but
   worth confirming with maintainers.

## Verdict

`merge-as-is`

Trivially correct one-line tuple extension with proportional
doc and test additions. The five tests cover the prefix match,
case sensitivity, multi-var forwarding, and isolation from
non-prefixed vars. Three small follow-ups can be threaded as
issues rather than blocking this PR: (a) a security note in the
doc comment about agent visibility of forwarded vars, (b) a
follow-up issue exploring a generic configurable allow-list to
avoid the per-vendor one-line PR pattern, and (c) replacing the
`'sk-test-key-12345'` test fixture with a non-secret-scanner-tripping
string. None of these block; the PR is a self-contained good
change.
