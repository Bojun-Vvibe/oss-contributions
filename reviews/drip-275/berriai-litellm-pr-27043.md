# Review: BerriAI/litellm #27043

- **PR:** BerriAI/litellm#27043 — `fix(security): sandbox jinja2 in gitlab/arize/bitbucket prompt managers`
- **Head SHA:** `de28a4f352bd5f1267973615e4ddbac6742630c2`
- **Files:** 4 (+152/-6)
- **Verdict:** `merge-as-is`

## Observations

1. **Correct fix and correct scope** — The three managers (`arize_phoenix_prompt_manager.py:77`, `bitbucket_prompt_manager.py:77`, `gitlab_prompt_manager.py:93`) all swap `jinja2.Environment(...)` for `jinja2.sandbox.ImmutableSandboxedEnvironment(...)` while preserving `loader=DictLoader({})`, `autoescape=select_autoescape(["html", "xml"])`, and the manager-specific delimiter customizations. The kwargs surface of the sandbox env is a strict superset for these uses, so behavior is identical for legitimate `{{ var }}` substitution.

2. **Threat model is real and underdocumented** — The inline comment on each manager explicitly calls out that anyone with repo write access (GitLab/BitBucket) or workspace edit access (Arize Phoenix) can ship `{{ ''.__class__.__mro__[1].__subclasses__() }}` style payloads to achieve RCE on the proxy host. This is a meaningful proxy-side hardening; the matching dotprompt manager already had this fix, so this PR closes the symmetry gap.

3. **Test design is solid** — `tests/test_litellm/integrations/test_prompt_manager_ssti.py` parametrizes 4 SSTI payloads × 3 managers = 12 SecurityError assertions, plus 3 instance-type assertions, plus 3 normal-render assertions. The factory pattern with `monkeypatch` to stub `BitBucketClient`/`ArizePhoenixClient` keeps the tests offline and fast (0.84s reported). The `_SSTI_PAYLOADS` list covers `__class__/__mro__/__subclasses__`, `__init__/__globals__/os.popen`, and `cycler.__init__.__globals__` — the canonical traversal chains.

4. **No behavioral change for trusted templates** — `ImmutableSandboxedEnvironment` blocks `__getattr__` traversal but leaves `{{ name }}`, `{% if %}`, `{% for %}`, filters etc. fully functional. The 137 existing prompt-manager tests passing confirms this. Worth noting the sandbox does not prevent unbounded loops or template-size DoS, but that's out of scope for an SSTI fix.

## Recommendation

Direct security fix with regression coverage. Merge.
