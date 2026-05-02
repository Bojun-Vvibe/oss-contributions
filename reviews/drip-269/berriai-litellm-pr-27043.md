# BerriAI/litellm #27043 — fix(security): sandbox jinja2 in gitlab/arize/bitbucket prompt managers

- URL: https://github.com/BerriAI/litellm/pull/27043
- Head SHA: `de28a4f352bd5f1267973615e4ddbac6742630c2`
- Author: @stuxf
- Stats: +152 / -6 across 4 files

## Summary

Closes a SSTI hardening gap: the dotprompt manager was previously moved to
`jinja2.sandbox.ImmutableSandboxedEnvironment`, but the three sibling
managers (gitlab, arize, bitbucket) still used a plain `jinja2.Environment`
where `{{ ''.__class__.__mro__[1].__subclasses__() }}` and friends would
pivot to RCE on the proxy host once an attacker has repo/workspace write
access. This PR brings the three managers in line and adds a parametrized
regression test suite locking in the sandbox + classic SSTI payload
rejection.

## Specific feedback

- `litellm/integrations/arize/arize_phoenix_prompt_manager.py:8-9` and
  `:78-87` — replaces `Environment` with `ImmutableSandboxedEnvironment`,
  preserves `loader=DictLoader({})`, `autoescape=select_autoescape(["html",
  "xml"])`, and the Mustache/Handlebars-style delimiter overrides.
  Behavior for legitimate `{{ var }}` substitution is unchanged.
- `litellm/integrations/bitbucket/bitbucket_prompt_manager.py:8-9` and
  `:78-87` — same swap, same comment block. Identical pattern.
- `litellm/integrations/gitlab/gitlab_prompt_manager.py:7-8` and
  `:94-103` — same swap. The inline justification ("anyone with repo write
  access … pivot into RCE on the proxy host") is the right
  threat-model note to leave at the call site.
- `tests/test_litellm/integrations/test_prompt_manager_ssti.py:1-125` —
  18-case parametrized suite covers (a) `isinstance(jinja_env,
  ImmutableSandboxedEnvironment)` for each manager, and (b) four
  attribute-traversal payloads raising `SecurityError`. The MagicMock /
  monkeypatch shims correctly stub the BitBucket/Arize clients so no
  network is touched. Test file lives in the right place under
  `tests/test_litellm/integrations/`.
- The PR body cites running `uv run pytest tests/test_litellm/integrations/
  test_prompt_manager_ssti.py -q` (18 passed) and the wider
  `{bitbucket,gitlab,arize}` directories (137 passed). Good evidence of
  no behavioral regression on the legitimate render path.
- Note: `ImmutableSandboxedEnvironment` is the right choice here (vs the
  weaker `SandboxedEnvironment`) because attackers can also abuse mutable
  collection methods (e.g. `dict.pop`) to alter shared state across
  renders. Worth keeping that rationale in the comment if anyone later
  weighs "downgrade to SandboxedEnvironment for x feature".

## Verdict

`merge-as-is` — security regression closure with thorough regression
tests. Comments are clear and consistent across files.
