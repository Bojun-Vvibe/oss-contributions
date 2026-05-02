# BerriAI/litellm PR #27043 — fix(security): sandbox jinja2 in gitlab/arize/bitbucket prompt managers

- Repo: `BerriAI/litellm`
- PR: #27043
- Head SHA: `de28a4f352bd5f1267973615e4ddbac6742630c2`
- Author: stuxf

## Summary
`DotpromptManager` was previously hardened to render templates through
`jinja2.sandbox.ImmutableSandboxedEnvironment`, but three sibling managers
(GitLab, Arize Phoenix, BitBucket) still used plain `jinja2.Environment()`,
keeping a SSTI → RCE path open for anyone with repo write or workspace
edit access. This PR swaps each one to `ImmutableSandboxedEnvironment` and
adds a 18-case regression suite.

## Specific references
- `litellm/integrations/gitlab/gitlab_prompt_manager.py:7-8,93-100` — drop
  `Environment` import, add `from jinja2.sandbox import
  ImmutableSandboxedEnvironment`, replace `Environment(...)` with the
  sandboxed variant. Comment correctly identifies the threat model
  (repo-write → RCE via `__class__.__init__.__globals__`).
- `litellm/integrations/bitbucket/bitbucket_prompt_manager.py:7-8,77-84`
  — identical treatment. Same threat model (repo-write users).
- `litellm/integrations/arize/arize_phoenix_prompt_manager.py:7-8,77-86`
  — identical treatment. Threat model is workspace-edit users in Arize
  Phoenix.
- `tests/test_litellm/integrations/test_prompt_manager_ssti.py:1-125`
  (new file):
  - `_SSTI_PAYLOADS` (line 26-31): four canonical attribute-traversal
    payloads (`__class__.__mro__[1].__subclasses__()`,
    `config.__class__.__init__.__globals__['os'].popen('id').read()`,
    `cycler.__init__.__globals__.os.popen('id').read()`,
    `().__class__.__bases__[0].__subclasses__()`).
  - `test_jinja_env_is_sandboxed` (line 81): asserts each `jinja_env` is
    an `ImmutableSandboxedEnvironment` instance.
  - `test_jinja_env_blocks_ssti_payloads` (line 100): each payload must
    raise `jinja2.exceptions.SecurityError` on render.

## Verdict
`merge-as-is`

## Rationale
This is a defensible-by-default security hardening with three properties
that make it an obvious accept:
1. The fix is the same one the dotprompt manager already uses, so there
   is no novel mitigation to second-guess. Behavioral parity with an
   already-hardened sibling is the right bar.
2. The test suite pins both the *type* of environment (instance check,
   so a future `Environment()` regression fails immediately) AND the
   *behavior* (SecurityError raise on canonical payloads). That's the
   correct "belt and suspenders" pattern for security regressions.
3. `ImmutableSandboxedEnvironment` is a strict superset of `Environment`
   for normal `{{ var }}` substitution — the test
   `test_jinja_env_is_sandboxed` is parametrized across all three
   managers, but does not need a "normal rendering still works" twin
   because the four payloads chosen are precisely the ones that *should*
   start raising; ordinary template rendering is exercised by the
   existing 137-test integration suites for each manager (per the PR
   body).

The custom delimiter configs (`{{`/`}}` for GitLab, Handlebars-style for
BitBucket/Arize) are preserved in each `ImmutableSandboxedEnvironment`
constructor call. Confirmed in diff.

No banned strings; security-sensitive change with thorough coverage. Ship.
