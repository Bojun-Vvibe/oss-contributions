# BerriAI/litellm #26663 — fix(auth): enforce access_group_ids model restriction when key/team models is empty

- **Head SHA**: `73bd597850aed9f7751131655103b5aa80d83378`
- **State**: OPEN
- **Author**: octo-patch
- **Size**: +385 / -6 across 6 files

## Files changed

- `litellm/proxy/auth/auth_checks.py` (+38/-5) — the *titled* fix
- `litellm/proxy/db/db_spend_update_writer.py` (+9/-0) — bonus team-key spend bug fix
- `litellm/llms/zai/chat/transformation.py` (+43/-1) — bonus ZAI/GLM content-flatten fix
- `tests/test_litellm/proxy/auth/test_auth_checks.py` (+114/-0)
- `tests/test_litellm/llms/zai/test_zai_provider.py` (+114/-0)
- `tests/test_litellm/proxy/db/test_db_spend_update_writer.py` (+67/-0)

## What it does

Three logically distinct fixes bundled in one PR, only one of which the title describes. **Reviewing all three because they all ship together if this lands.**

1. **The titled fix** — closes `#26656`. A key or team configured with `access_group_ids = ["embeddings"]` and `models = []` was incorrectly granting all-model access. Root cause is order of operations in `can_key_call_model` / `can_team_access_model`: `_can_object_call_model` evaluated `len(models) == 0` as "all-model access — return True" *before* the `except ProxyException` fallback got a chance to inspect `access_group_ids`. The check was structurally a fallback when in fact it should have been a precondition.
2. **An undeclared bonus** — `db_spend_update_writer.py` adds a `team_id` parameter and a `if team_id is not None: return` early-exit guard in `_update_user_db` so calls made via team keys no longer accumulate spend on `LiteLLM_UserTable.spend`. The comment says this prevents false `BudgetExceededError` on personal key calls when the Redis cross-pod counter is missing (after a restart).
3. **A second undeclared bonus** — ZAI provider gets a new `_transform_messages` override that flattens OpenAI multi-part `content: [{"type": "text", "text": "..."}]` to a plain string for `tool` and `assistant` messages, because GLM's Jinja chat template does `m.content is string` and silently drops list-format content (issue `#25868`).

## Specific observations

### Auth fix (the titled one)

- **The reordering at `auth_checks.py:2695-2710` (key path) and `:2770-2787` (team path) is the right shape.** Pulls `key_access_group_ids = valid_token.access_group_ids or []` out of the `except` block to the top of the function, then early-returns through the access-group-resolved path *before* the `try` if the precondition `key_access_group_ids and not valid_token.models` holds. This converts the original "fallback when denied" into a "precondition when partially configured" — exactly the right semantic for "user explicitly named a constraint, honor it instead of defaulting to all-access."
- **The original `except ProxyException` fallback path is preserved unchanged** for the case where `models` is non-empty but the model isn't in that list and the access-groups can grant access on top. Two distinct execution paths now exist (`models=[]` precondition vs. `models=[...]` fallback) and they don't fight each other. Good.
- **The 4-test matrix at `test_auth_checks.py:1-114` covers exactly the right cells**: (key, team) × (model in group, model not in group), all four with `models=[]` to exercise the new precondition path. **Missing cells** that would round out coverage: `models=["model-a"]` + `access_group_ids=["group-x"]` where the model is in neither (current fallback denies — should still pass), where the model is in group but not in models (current behavior?), and where `access_group_ids=[]` and `models=[]` (should it grant all-access? — the PR doesn't change this but the test would pin "no regression on the most-permissive cell").
- **Subtle invariant the PR does not state**: an empty `access_group_ids` no longer matters for the precondition path because the guard is `if key_access_group_ids and not valid_token.models`. A token with `access_group_ids=[], models=[]` still falls through to the legacy `_can_object_call_model` which returns "all-model access." This is presumably intentional (only configured access groups should constrain), but worth a one-line comment or test pinning that "all-empty stays all-access."

### Spend writer fix (undeclared bonus)

- **The `team_id is not None: return` early-exit at `db_spend_update_writer.py:511-518` is correct in shape but the rationale embedded in the comment doesn't fully match the threading.** The comment says "Skip personal spend tracking for team key calls. Team spend is tracked separately via `_update_team_db`." That's the right policy. **The threading concern**: `_update_user_db` is called from `_batch_database_updates` at `:330-340`, and the new `team_id=team_id` argument is plumbed in. If *any* other caller of `_update_user_db` exists in the codebase that doesn't pass `team_id`, those callers now fail open (default `team_id=None` → no early exit → spend still accumulates) which is the original buggy behavior. Worth a `git grep '_update_user_db'` to verify only the one caller exists.
- **The 67-line test in `test_db_spend_update_writer.py` likely covers the new branch**, but I haven't seen the test body in the diff snippet — would want to see it explicitly assert that with `team_id="team-x"` the user's spend row is *not* updated, and with `team_id=None` it *is*.
- **Not in PR scope but worth raising**: the bug as described ("spend accumulates on user even for team-key calls") may be load-bearing somewhere else — for instance, an admin dashboard "user spend" panel may have been over-reporting and someone may be relying on those over-reports. Behavior change deserves a release-note callout.

### ZAI content-flatten fix (undeclared bonus)

- **`_flatten_content_parts` at `zai/chat/transformation.py:11-31` correctly handles the four cases**: `str`/`None` pass through, `list` of `{"type": "text", "text": "..."}` parts get joined with `\n`, list of bare strings join too, fallback returns `content` as-is. **`if text:` at `:24` is silently lossy** — empty-string `text=""` parts get dropped. For the GLM chat-template bug this is right (the template only consumes truthy strings anyway), but if a future tool-result intentionally returns an empty-string acknowledgment, that signal vanishes. A comment naming this would prevent surprise.
- **`_transform_messages` at `:84-100` only flattens for `role in ("tool", "assistant")`** — user messages with multi-part content are left alone, which is correct because GLM's template handles user multi-part fine; only tool/assistant got broken. The match against the issue (`#25868`) is precise.
- **`# type: ignore` at `:99-100`** marks the `super()._transform_messages` call as type-ignored. The signature mismatch is between the parent's expected `List[AllMessageValues]` and the union return shape `Union[List[AllMessageValues], Coroutine[...]]`. The `is_async=False` path returns sync, the `is_async=True` returns coroutine — this is a real type-system limitation, not a sloppy ignore. A `@overload` at the class level would let the type system express the conditional return shape and remove the ignore, but that's a broader refactor than this PR.
- **Test at `test_zai_provider.py` covers tool and assistant cases separately**, which pins both branches of the role-filter at `:96-99`. Good.

### Cross-cutting concerns

- **One PR, three unrelated fixes** is the biggest issue. Each fix has its own root cause, its own test, its own blast radius. Bisecting a regression that lands after this merge would be much harder than if these were three PRs. Reviewer load is also tripled — a maintainer reviewing the auth fix has no way to know whether the spend writer change is in scope. Recommend splitting before merge if at all possible.
- **PR description only describes fix #1**. Fixes #2 and #3 are not mentioned. A maintainer doing a quick approve based on the description would silently approve two undeclared changes.
- **Mixed-domain test files** — auth tests touch auth, spend tests touch spend, ZAI tests touch ZAI — which means the test surface at least mirrors the code surface. Good hygiene given the bundled shape.

## Verdict

**request-changes** — three logically independent fixes bundled into one PR with the description only naming one of them. The titled auth fix itself is correct in shape with a coherent 4-test matrix and the right precondition-vs-fallback restructuring; the bonus spend-writer and ZAI fixes look correct in isolation but ship undisclosed. Strongly recommend splitting into three PRs so each can be reviewed against its own issue, blast radius, and test coverage. If splitting is genuinely off the table, at minimum: (a) PR description must enumerate all three fixes with their root causes, (b) a `git grep '_update_user_db'` audit must confirm no caller-side regression risk for the new `team_id` parameter, (c) the auth fix needs a test for the all-empty case to pin "no constraint stays all-access."
