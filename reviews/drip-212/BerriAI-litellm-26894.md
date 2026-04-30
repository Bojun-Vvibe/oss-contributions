# BerriAI/litellm PR #26894 — [Fix] Remove unwanted metadata info from LangSmith

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/26894
- Head SHA: `aa62ed2d3cdb99fcd5acb65bfc02a4e606dd90f6`
- State: OPEN
- Files: 2 (`litellm/integrations/langsmith.py`, `tests/test_litellm/integrations/test_langsmith_init.py`), +94/−1
- Verdict: **merge-after-nits**

## What it does

Honors the global `litellm.redact_user_api_key_info` flag in the LangSmith
integration. Before, `LangsmithLogger._build_extra_metadata(metadata)` at
`litellm/integrations/langsmith.py:154-156` lifted any
`{session_id, thread_id, conversation_id}` out of `requester_metadata`
into `extra_metadata` but did **not** scrub the `user_api_key_*` family
of fields, so LangSmith traces always carried `user_api_key_hash`,
`user_api_key_alias`, `user_api_key_user_id`, `user_api_key_team_id`,
`user_api_key_team_alias`, `user_api_key_request_route`, and
`user_api_key_spend` — both at the top level and nested under
`requester_metadata`. The fix calls
`redact_user_api_key_info(metadata=extra_metadata)` once on the
top-level dict and then, when the flag is on, recurses one level into
the nested `requester_metadata` dict (since the helper is shallow):

```python
extra_metadata = redact_user_api_key_info(metadata=extra_metadata)
if litellm.redact_user_api_key_info is True:
    nested = extra_metadata.get("requester_metadata")
    if isinstance(nested, dict):
        extra_metadata["requester_metadata"] = redact_user_api_key_info(
            metadata=nested
        )
```

The 78-line test class `TestLangsmithRedactUserApiKeyInfo` covers
3 cases: flag-off-preserves-fields, flag-on-strips-top-level, and
flag-on-strips-nested-but-keeps-lifted-session-id. A `reset_redact_flag`
fixture restores `litellm.redact_user_api_key_info` between tests.

## What's right

- The shape — "scrub at the LangSmith handler boundary, not in the
  shared metadata builder" — is correct. The `user_api_key_*` fields
  are deliberately threaded through the proxy plumbing and a global
  scrub at the source would break cost/usage attribution paths that
  legitimately consume them. Doing it at the third-party-integration
  boundary is the minimum-blast-radius spot.
- The "shallow helper, so re-apply to nested dict" comment at
  `langsmith.py:156-157` is honest about the limitation and is the
  right way to document why the second call exists. Without that
  comment the next reader would assume the helper is recursive and
  either delete the duplicated call or wonder why it's there.
- Three tests cover both polarities of the flag and both nesting
  levels, plus the "lifted session_id is preserved at top level" path
  (line 130: `extra["session_id"] == "sess-1"`) — that's the right
  invariant to pin since the lift happens *before* the scrub at
  `_build_extra_metadata`, so the lifted copy survives.
- The `reset_redact_flag` fixture (line 42-47) yields-then-restores the
  global, which is the correct shape for module-state tests under
  pytest-xdist.

## Nits (request before merge)

1. **Asymmetric flag check.** Top-level redaction calls
   `redact_user_api_key_info(metadata=extra_metadata)` *unconditionally*
   (the helper is presumably a no-op when the flag is off), but the
   nested redaction is gated by an explicit `if
   litellm.redact_user_api_key_info is True:`. Either both should be
   gated (faster early-out), or both should be unconditional (uniform
   shape, fewer branches). The asymmetry is confusing — pick one and
   document.
2. **`is True` instead of truthy.** `if litellm.redact_user_api_key_info
   is True:` is stricter than the rest of the codebase's typical
   `if litellm.redact_user_api_key_info:` idiom. If the flag is ever
   set to a truthy non-`True` value (e.g. `"1"`, `1`) by an env-var
   parser, the nested scrub will silently skip while the top-level
   scrub still happens (assuming the helper itself does truthy check).
   Use plain truthy or document why strict identity is required.
3. **No regression test for flag-off + no `requester_metadata`.** The
   existing 3 tests all use a metadata dict with `requester_metadata`
   present. Add a 4th case where `requester_metadata` is absent or
   non-dict to pin the `isinstance(nested, dict)` guard. Cheap, and
   prevents the next refactor from "tidying" the guard away.
4. **Mutation vs. copy.** The new code rebinds
   `extra_metadata = redact_user_api_key_info(metadata=extra_metadata)`,
   which suggests the helper returns a new dict rather than mutating in
   place. But then the nested call assigns
   `extra_metadata["requester_metadata"] = redact_user_api_key_info(
   metadata=nested)` — also a fresh dict. If the helper happens to
   mutate-and-return-self, the original `metadata` argument passed
   *into* `_build_extra_metadata` could be affected. Worth confirming
   `redact_user_api_key_info` is non-mutating (or doing a defensive
   `dict.copy()` at the call site). The test currently builds a fresh
   dict from `_metadata_with_user_api_key_fields()` per test, so it
   would not catch cross-call mutation.
5. **Test name nit.** `test_redact_enabled_strips_nested_requester_metadata`
   asserts both nested stripping and top-level `session_id` lift; the
   second assertion is a bonus invariant orthogonal to the test name.
   Either rename or split.

## Why merge-after-nits

The fix is at the right boundary, the test coverage is honest about
both polarities of the flag and both nesting levels, and the
"shallow helper, recurse manually" comment names the constraint. The
nits are about API-shape consistency (`is True` vs. truthy, gated vs.
ungated calls) and one missing edge-case test, not correctness —
none of them block, but landing them all keeps the contract clean for
the next maintainer who has to integrate a third-party logger that
also needs to honor this flag.
