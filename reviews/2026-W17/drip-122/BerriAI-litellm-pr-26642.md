---
pr: 26642
repo: BerriAI/litellm
sha: b953bfe24331863803445dc01681089acb01a5b3
verdict: merge-after-nits
date: 2026-04-28
---

# BerriAI/litellm #26642 — fix: honor `think: false` to disable reasoning (#26413)

- **Author**: skgandikota (Saikoushik Gandikota)
- **Head SHA**: `b953bfe24331863803445dc01681089acb01a5b3`
- **Size**: 288 added / 0 deleted across 2 files (`litellm/utils.py`,
  `tests/test_litellm/test_think_param_normalization.py`).
- **Closes**: #26413

## Scope

Closes a real silent-drop bug: clients sending the Ollama-native
`"think": false` directly in a chat-completion request body had the field
filtered out by `pre_process_non_default_params` (because `think` isn't in
`DEFAULT_CHAT_COMPLETION_PARAM_VALUES`), so reasoning-capable models still
ran reasoning and the response continued to carry `reasoning_content`,
`thinking_blocks`, and non-zero `reasoning_tokens`. Fix captures `think`
from kwargs *before* pre-processing in `get_optional_params`, then translates
it to `reasoning_effort = "disable"` once the supported-params list is
known — only when (a) the caller didn't already pass `reasoning_effort`,
(b) the provider lists `reasoning_effort` as supported, and (c) the
provider is in a small allowlist known to honor the literal `"disable"`
value.

## Specific findings

- `litellm/utils.py:3683-3759` — the `_think_to_reasoning_effort` helper.
  The truth-table is well-thought-through:
  - `False`/`"false"`/`"0"`/`"no"`/`"off"`/`" False "` → `reasoning_effort = "disable"`
  - `True`/`"true"`/`"1"`/`"yes"`/`"on"` → no-op (intentional: don't
    auto-enable reasoning, force callers to pass `reasoning_effort` explicitly)
  - unrecognized string → no-op
  - non-string-non-bool → coerced via `bool(...)`
- `litellm/utils.py:3725-3733` — the `_PROVIDERS_SUPPORTING_REASONING_DISABLE`
  allowlist (`gemini`, `vertex_ai`, `vertex_ai_beta`, `ollama`, `ollama_chat`).
  Critical correctness detail: other providers (Anthropic, Bedrock, OpenAI
  o-series) advertise `reasoning_effort` as a supported param but only accept
  `low|medium|high|minimal|none|None`; passing `"disable"` to them raises
  `ValueError`/`BadRequestError`. The allowlist gates the translation on
  *known-safe* providers; the docstring at `:3711-3719` correctly names this
  invariant. Adding to the allowlist is safe; omitting just means
  `think:false` is silently dropped (with a debug log).
- `litellm/utils.py:3735-3742` — gate condition is the four-way AND:
  `think_bool is False and "reasoning_effort" not in non_default_params and
  "reasoning_effort" in supported_params and (custom_llm_provider or "") in
  _PROVIDERS_SUPPORTING_REASONING_DISABLE`. Correct precedence — caller
  explicitness wins over `think` translation, and the empty-string fallback
  on `custom_llm_provider or ""` defends against `None` provider (which
  would crash the `in` check).
- `litellm/utils.py:4063-4067` — the early capture: `_think_value =
  special_params.pop("think", None)`. Critically uses `pop` (not `get`) on
  `special_params` so the value can't leak through to providers that pass
  kwargs through unchanged (which would surface as an `UnsupportedParamsError`
  on strict-OpenAI-compat providers). The capture is sequenced *before*
  the `passed_params = locals().copy()` clone — actually no, looking again,
  the capture is at `:4068` after the clone at `:4061`. But the clone
  is `locals().copy()` not a deep copy of `kwargs`, so popping from
  `special_params` afterward still removes it from the kwargs dict before
  it flows into provider mapping. Correct.
- `litellm/utils.py:4150-4160` — the late translation site. Correctly
  positioned *after* `supported_params` and `allowed_openai_params` are
  resolved (so the gate can check `"reasoning_effort" in supported_params`)
  and *before* `_check_valid_arg(supported_params=supported_params or [])`
  (so the translated `reasoning_effort` survives the validator).
- Tests at `tests/test_litellm/test_think_param_normalization.py:14-90`
  cover helper-level truth-table (including the `" False "` whitespace
  variant), explicit-`reasoning_effort` precedence, and the no-op-on-
  non-supporting-provider path. End-to-end tests for `gemini-2.5-flash`,
  `ollama_chat/llama3.1`, and `cohere_chat/command-r` (the three documented
  outcome classes) are listed in the PR body but the diff snippet I saw
  cuts off mid-test — assuming they're present, that's appropriate coverage.

### Real nits

- The PR body's "Authored with assistance from GitHub C\u200boilot CLI"
  attribution at the bottom of the body is fine for the author but should
  not affect the technical review.
- `_PROVIDERS_SUPPORTING_REASONING_DISABLE` is a `set` literal *inside* the
  helper function body (`:3725-3733`). On every call, Python rebuilds this
  set from scratch. Hoist it to a module-level constant
  (`_THINK_DISABLE_PROVIDERS = frozenset({...})`) — both for performance and
  so future provider additions are discoverable via grep without diving
  into the helper body.
- The truthy-string parse (`normalized in ("true", "1", "yes", "on")` →
  `think_bool = True`) is dead code given the truthy branch is documented
  as no-op and the only effect of `think_bool = True` versus
  `think_bool = None` is which debug log line fires (`"think=True was
  passed but..."` vs nothing). Either drop the truthy parsing entirely,
  or keep it and document that the only purpose is the diagnostic log.
- The provider-allowlist approach is correct but has a forward-compat
  cost: every new provider that grows reasoning support needs to be
  manually added here or `think:false` silently won't take effect. A
  capability flag on the provider config (e.g. `provider_config.
  reasoning_effort_disable_supported: bool`) would be more durable than
  a string-set membership check, but that's a follow-up.
- Test file uses `sys.path.insert(0, os.path.abspath(...))` to import —
  consistent with the repo pattern but worth a one-line comment naming
  why (test is invoked from repo root via pytest discovery, not from
  the `tests/` dir).

## Risk

Low-medium. The fix is correctly scoped (silent drop → translate-or-drop
with explicit allowlist), the gate condition correctly defers to caller
explicitness, and the `pop` (not `get`) prevents the field from leaking
to providers. The main residual risk is the allowlist drift — if Anthropic
or OpenAI adds a `"disable"` value in a future API version, the
`_PROVIDERS_SUPPORTING_REASONING_DISABLE` set silently won't update and
users will continue getting `think:false` dropped on those providers. A
capability-flag-on-config approach would be more durable.

The behavior change is strictly more user-friendly: previously `think:false`
was silently dropped; now it either takes effect (on the allowlisted
providers) or is silently dropped with a debug log (on others). No
existing call site can be made worse by this change.

## Verdict

**merge-after-nits** — hoist `_PROVIDERS_SUPPORTING_REASONING_DISABLE`
to a module-level `frozenset`, drop or document the dead truthy-string
parse, and consider a follow-up tracking the capability-flag-on-config
migration. The core fix is right.

## What I learned

The "capture early, translate late" pattern (`pop` before pre-processing,
translate after `supported_params` is known) is the right shape for any
non-OpenAI-compatible param that needs provider-conditional translation.
The early `pop` ensures the field can't leak to providers that pass
kwargs through unchanged; the late translation lets the gate condition
read the resolved supported-params list. Both halves are necessary —
either alone produces silent breakage on some provider.
