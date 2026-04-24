# PR #26400 — deduplicate overlapping keys between `kwargs` and `optional_params` in websearch agentic loop

**Repo:** BerriAI/litellm • **Author:** jesset • **Status:** open
• **Net:** +154 / −0

## Change

In `litellm/integrations/websearch_interception/handler.py`, both
`_build_anthropic_request_patch` and
`_build_chat_completion_request_patch` now strip from
`kwargs_for_followup` any key already present in
`optional_params_*` before returning the `AgenticLoopRequestPatch`:

```python
kwargs_for_followup = {
    k: v
    for k, v in kwargs_for_followup.items()
    if k not in optional_params_without_max_tokens
}
```

Two regression tests added (one per request-builder path) that pass
overlapping `temperature` / `top_p` / `stop` in both dicts and
assert (a) overlap is removed from kwargs, (b) `optional_params`
values are preserved, (c) non-overlapping kwargs survive, (d)
internal flags (`litellm_logging_obj`, `_websearch_interception_*`)
are still filtered.

## Root cause analysis

The follow-up call site unpacks both dicts via `**`:

```python
await acompletion(**optional_params, **request_patch.kwargs)
```

When the same key (`temperature`) lives in both, Python raises:

```
TypeError: acompletion() got multiple values for keyword argument 'temperature'
```

— and the entire agentic loop dies mid-turn. The user sees a
generic `TypeError`, not a "websearch interception layer has a
duplicate-key bug" message.

## Why the fix is correct but the policy choice is implicit

The dedup keeps `optional_params` and drops from `kwargs`. The
test comment says "optional_params takes precedence" — that's the
right call because:

1. `optional_params` is the curated, provider-mapped, validated
   set; `kwargs` is whatever the caller threw in.
2. Web-search interception modifies `optional_params` (e.g. injects
   `tools`); deferring to `optional_params` ensures those
   modifications stick.

But this precedence is **not documented** in the function's
docstring or type. A future contributor who reads
`_prepare_followup_kwargs` and doesn't read this comment might
wonder why their `kwargs` setting is silently ignored. A one-line
comment at the dedup site (`# optional_params is the curated source
of truth post-interception; kwargs only contributes keys not
already mapped`) would document the contract.

## Sharp edge: the dedup is one-shot, not transitive

The Anthropic path uses `optional_params_without_max_tokens` as the
exclusion set. The Chat Completion path uses `optional_params_clean`.
Both are derived dicts — built shortly before the dedup — that may
not match the *final* dict actually unpacked at the call site (if a
later code path adds keys to `optional_params`). If a key gets added
to `optional_params` after the dedup but before the unpack, it can
re-collide with `kwargs` and re-trigger the original bug.

Quick check: is there any code path between
`AgenticLoopRequestPatch` construction and the unpack that mutates
`optional_params`? If yes, the dedup needs to happen at the unpack
site, not at the patch-build site. If no (most likely), worth a
comment: `# Dedup is correct because optional_params is frozen
between here and the unpack site.`

## Test coverage assessment

Strong — both paths covered, both with overlapping numeric and
list-valued keys (`temperature`, `top_p`, `stop`). Internal-flag
filtering is asserted to still work post-dedup. The one gap: no
test for the case where a key is in `kwargs` but NOT in
`optional_params` — i.e., the no-op case. That's implicitly covered
by `assert plan.request_patch.kwargs["custom_param"] == "keep_me"`,
but a dedicated assertion would harden the contract.

## Risk

Very low. The dedup is a single dict comprehension on a small dict;
no perf concern. The behavioral change is "kwargs values that were
silently shadowed by `**optional_params` ordering before are now
explicitly removed" — but `**` unpacking with duplicates raises
`TypeError` in Python, so there's no scenario where kwargs values
were *being used* and are now lost. The only thing changing is the
error mode: from `TypeError` crash to silent acceptance.

## Verdict

Correct, well-tested fix for a real agentic-loop crasher. The
precedence policy is the right one. A two-line documentation
addition would harden it against future drift. Worth merging.

## What I learned

Whenever two dicts get `**`-unpacked into the same call, the
"who wins" question must be settled at construction time, not at
the call site — Python gives you a `TypeError`, not last-write-wins.
The dedup-at-construction pattern in this PR is the cleanest fix.
