# browser-use/browser-use#4734 — fix(schema): remove unreachable 'type' key from validation fields list

- **URL**: https://github.com/browser-use/browser-use/pull/4734
- **Author**: Will-hxw
- **Head SHA**: `c4f712b90b73cee6c493273e3d34946cbfbacc8e`
- **Verdict**: `merge-as-is`

## Summary

One-line dead-code removal in `browser_use/llm/schema.py:91` inside
`optimize_schema`. The `'type'` literal in the "essential validation
fields" `elif key in [...]` branch was unreachable because the
preceding branches in the same `optimize_schema` walk already
intercept and rewrite `'type'` keys before this check fires. Dropping
it has no behavioural change.

## Reviewable points

- The diff is exactly one removed line:
  ```
  -            'type',
                'required',
                'minimum',
                'maximum',
  ```
  Reachability claim is verifiable from the surrounding `optimize_schema`
  control flow: any dict carrying `type` is handled in the earlier
  `if key == 'type'` / `if 'type' in obj` branches that strip or
  normalise it before the `elif key in [...]` validation-keep list is
  consulted.
- Companion PR #4719 (same author) appears to remove the
  corresponding code path that produced the unreachable branch. Worth
  ordering the merges so #4719 lands first; otherwise this change is
  purely cosmetic.
- No test change is needed — the keep-list is consulted only for keys
  that are not `'type'`, so the assertion "removing `'type'` changes
  no observable output" holds.

## Rationale

Trivial cleanup of a literal that cannot fire. Low-risk, no behavioural
delta, no tests required. Safe to merge as-is provided #4719 is in
flight or already merged so the dead branch genuinely stays dead.
