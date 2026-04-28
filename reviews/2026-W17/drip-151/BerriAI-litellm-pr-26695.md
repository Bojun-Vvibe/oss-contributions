# BerriAI/litellm#26695 — fix(tools): resolve legacy `definitions` $ref in tool schemas for Anthropic and Fireworks

- **Repo:** [BerriAI/litellm](https://github.com/BerriAI/litellm)
- **PR:** [#26695](https://github.com/BerriAI/litellm/pull/26695)
- **Head SHA:** `f1ab7da89df8ea1e9cd063ea011b198ca540c1df`
- **Size:** +6284 / -1146 across 60+ files (load-bearing change is +40 / -0 across two files; rest is rebase/unrelated)
- **State:** OPEN
- **Closes:** #26692

## Context

Some MCP servers (DevRev cited) emit tool `input_schema` objects that use
the legacy JSON Schema draft-04 `definitions` keyword with `$ref` pointers
like `#/definitions/_gen:tags`. When this gets forwarded to Anthropic or
Fireworks, the request fails at the provider:

- **Anthropic**: `Error resolving schema reference '#/definitions/_gen:tags': PointerToNowhere`
- **Fireworks**: `AttributeError('NoneType' object has no attribute 'lookup')`

Root cause: Anthropic supports `$defs` (draft-2020) but not `definitions`
(draft-04). The `_allowed_properties` filter in
`_map_tool_helper` strips the `definitions` key but leaves the dangling
`$ref` pointers — provider gets a schema with refs and nothing to resolve
them against.

GPT models work because OpenAI resolves `$ref` server-side. So this is a
real provider-portability gap the MCP ecosystem is hitting.

## Design analysis

The actual fix is two small additions; the rest of the diff (60+ files)
appears to be unrelated rebase noise that should be cleaned up before
merge — see nit 1.

### Anthropic side

`litellm/llms/anthropic/chat/transformation.py:448-470` — added before
the existing `_allowed_properties` filter:

```python
if "definitions" in _input_schema:
    import copy
    from litellm.litellm_core_utils.prompt_templates.common_utils import (
        unpack_defs,
    )

    _input_schema = copy.deepcopy(_input_schema)
    defs: dict = {
        **_input_schema.pop("definitions", {}),
        **_input_schema.pop("$defs", {}),
    }
    if defs:
        unpack_defs(_input_schema, defs)
```

Three things this does right:

1. **Reuses `unpack_defs`** — the existing helper already used elsewhere
   in the file for `response_format` schema handling. One source of
   truth for ref-inlining logic.
2. **Merges `definitions` and `$defs` into one dict before unpacking** —
   so a schema that defensively includes both styles still resolves
   correctly. The merge order (definitions first, $defs second) means
   `$defs` wins on key collision, which matches "newer spec wins."
3. **Gates on `if "definitions" in _input_schema`** — only pays the
   `deepcopy` cost when the legacy keyword is present. Preserves the
   hot-path performance for the 99% case.

The `deepcopy` at `:454` is necessary because `unpack_defs` mutates in
place and the original schema may be reused (cached, retained by
caller). Right call.

### Fireworks side

`litellm/llms/fireworks_ai/chat/transformation.py:200-223` — same
pattern in `_transform_tools`:

```python
params = tool["function"].get("parameters")
if params and "definitions" in params:
    from litellm.litellm_core_utils.prompt_templates.common_utils import (
        unpack_defs,
    )
    params = copy.deepcopy(params)
    defs: dict = {
        **params.pop("definitions", {}),
        **params.pop("$defs", {}),
    }
    if defs:
        unpack_defs(params, defs)
    tool["function"]["parameters"] = params
```

Symmetric to Anthropic side. Slight asymmetry: this one mutates the
input `tools` list (writes back `tool["function"]["parameters"] = params`),
the Anthropic side rebinds `_input_schema` and continues with the local
variable. Both work but the Anthropic version is cleaner because the
caller doesn't share the schema reference.

### Tests

The PR description claims three new tests:
- `test_anthropic_tool_with_legacy_definitions_ref_resolved_inline`
- `test_anthropic_tool_defs_ref_still_preserved`
- `test_fireworks_transform_tools_resolves_definitions_refs`

The middle one is the regression guard — `$defs` (which Anthropic
handles natively) must NOT be touched. That's the test that will catch
"someone tightens the unpack to `$defs` too" regressions.

## Risks / nits

1. **PR scope sprawl.** Touching 60+ files with +6284 / -1146 dwarfs the
   actual fix (+40 / -0 in two files). The diff includes guardrail
   hooks, predibase handler refactors, UI test deletions
   (`SpendLogsSettingsModal.test.tsx` deleted entirely, -484 lines),
   xecguard new feature (+1904 lines of tests), `model_prices_and_context_window.json` line reordering, and dozens of other unrelated
   changes. **Strongly request: rebase onto current main and force-push
   to keep only the schema-ref fix.** Without that, CI signal is muddy
   and reviewers can't be sure nothing else regressed.

2. **`unpack_defs` failure modes not enumerated.** If the schema has a
   circular reference (`#/definitions/A` → `#/definitions/B` →
   `#/definitions/A`), does `unpack_defs` infinite-loop, raise, or
   silently drop? Worth a one-line test for the circular case. The
   existing `response_format` callsite probably never hits this because
   the schemas there are author-controlled, but MCP-emitted schemas are
   not.

3. **Silent drop on missing ref target.** If a schema has `$ref:
   "#/definitions/Foo"` but `definitions` doesn't contain `Foo`, what
   does `unpack_defs` do? If it leaves the ref intact, we're back to
   the original "PointerToNowhere" error but routed through litellm
   instead of the provider — same UX. If it nulls the slot, we silently
   ship a malformed schema. Either way, the error message at the user
   should name *litellm* and *the missing ref* rather than the
   provider's opaque error. A try/except wrapper that raises a clear
   `LitellmError` would help.

4. **Performance on large schemas.** `deepcopy` of a 100-tool MCP
   server's combined schemas could be measurable on every request. If
   the same tool list comes through repeatedly in a chat session,
   memoization keyed on `id(tool["function"]["parameters"])` would
   amortize. Not a blocker, but a profile-worthy follow-up.

5. **Bedrock / Vertex / other Anthropic-derived paths not covered.**
   The PR fixes the direct Anthropic and Fireworks paths, but Bedrock's
   Anthropic Claude transformation (`litellm/llms/bedrock/messages/invoke_transformations/anthropic_claude3_transformation.py`) shows up in the diff and *might* hit the same issue. The PR
   doesn't add a `definitions` resolver there. If it shares the
   downstream Anthropic API, it has the same bug. Worth confirming.

## Verdict

**request-changes.** The fix itself is correct and well-scoped, but the
PR scope (nit 1) is unmergeable as-is — 60+ unrelated files in a single
PR is a CI signal disaster and a `git blame` future-tax. Pre-merge
ask: rebase to keep only the two transformation file changes plus the
three claimed tests. Once that's done, this is a `merge-after-nits` on
items 3 and 5.

## What I learned

- The split between draft-04 `definitions` and draft-2020 `$defs` is a
  real interop tax in the MCP world — server authors emit either, and
  every downstream provider has its own preference. A litellm-side
  resolver inlining refs to a flat schema is the right place to absorb
  that complexity (one fix benefits N providers).
- Reusing `unpack_defs` from `common_utils` instead of writing a new
  ref-resolver is the right call. JSON Schema ref resolution is a
  rabbit hole (relative refs, fragments, anchors, JSON Pointer escape
  rules); the existing helper has presumably been hardened.
- Scope discipline matters more for review quality than for any single
  technical detail. A 40-line fix wrapped in a 6000-line rebase becomes
  unreviewable not because the fix is wrong but because the reviewer
  can't be confident *what else* changed.
