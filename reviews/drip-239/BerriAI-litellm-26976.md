# BerriAI/litellm#26976 — feat(embedding): default OpenAI-path encoding_format to float

- **PR**: https://github.com/BerriAI/litellm/pull/26976
- **Author**: Sameerlite
- **Head SHA**: `5c72a95289a0b342fc0e5a9a7b379a9095a41042`
- **Files**: `litellm/main.py` (+5 / -2), `tests/local_testing/test_embedding.py` (+8 / -22), `tests/test_litellm/test_openai_embedding_encoding_format_default.py` (new, +86), `litellm/llms/hosted_vllm/embedding/README.md` (+12 / -1)
- **Verdict**: **needs-discussion**

## Context

The pre-fix `embedding(...)` function in `litellm/main.py` had a load-bearing comment at `:4923-4926`:

```python
if encoding_format is not None:
    optional_params["encoding_format"] = encoding_format
else:
    # Omiting causes openai sdk to add default value of "float"
    optional_params["encoding_format"] = None
```

The semantics: when the caller did *not* pass `encoding_format`, litellm explicitly set it to `None` and forwarded `encoding_format=None` to the OpenAI SDK. The comment claims this was load-bearing — that omitting the field would cause the OpenAI SDK to add `'float'` as a default automatically (the prior committed test text `test_encoding_format_none_not_omitted_from_openai_sdk` says the SDK actually defaults to `'base64'`, contradicting the comment).

The companion docstring on the deleted test at `tests/local_testing/test_embedding.py` (pre-fix `:1257-1273`) explained the *real* reason for the explicit `None`:

> Without this fix:
> - OpenAI SDK adds `encoding_format='base64'` as default when parameter is missing
> - This causes issues with providers that don't support `encoding_format` (like Gemini)
>
> With this fix:
> - `encoding_format=None` is explicitly passed
> - OpenAI SDK respects the explicit `None` and doesn't add defaults

So the prior code was specifically engineered to send `None` to the SDK to *suppress* a `'base64'` default that would break non-OpenAI providers.

## What's new

The PR replaces the explicit-`None` path with a four-tier resolution chain:

```python
# main.py:4923-4929
if encoding_format is not None:
    optional_params["encoding_format"] = encoding_format
else:
    optional_params["encoding_format"] = (
        optional_params.get("encoding_format")
        or get_secret_str("LITELLM_DEFAULT_EMBEDDING_ENCODING_FORMAT")
        or "float"
    )
```

Resolution order: explicit arg → existing `optional_params.encoding_format` → env var → hardcoded `"float"`. The deleted test is replaced by `test_encoding_format_defaults_to_float_for_openai_sdk` (which now asserts `call_kwargs["encoding_format"] == "float"`) plus a new dedicated file `tests/test_litellm/test_openai_embedding_encoding_format_default.py:+86` covering the env-var override and explicit-arg-overrides-env cases.

## What's right (about the fix shape)

- **Resolution chain order is correct.** Explicit > optional_params > env > hardcoded default. This matches the standard config-precedence convention everywhere else in litellm.
- **`get_secret_str("LITELLM_DEFAULT_EMBEDDING_ENCODING_FORMAT")`** is the right primitive (it handles the secrets-manager indirection, not just `os.environ`), and the env-var name follows the `LITELLM_DEFAULT_*` convention.
- **README documentation at `litellm/llms/hosted_vllm/embedding/README.md:+12`** correctly documents the four-tier resolution chain in the user-facing format. The env-var-name + the precedence order + the rationale ("avoids forwarding `encoding_format=None` to the provider/SDK where some servers behave poorly") are all in the docs.
- **Test coverage of the new resolution chain is solid.** `test_openai_embedding_encoding_format_default.py:9-49` parametrizes `(set_env=False) → "float"` and `(set_env=True, env_value="base64") → "base64"`. `test_openai_embedding_encoding_format_explicit_overrides_env` at `:51-86` pins the precedence rule. Both use the `_get_openai_client` patching pattern already established in the codebase.

## Why this is needs-discussion, not merge-as-is

**The PR re-introduces the exact bug that the deleted test was guarding against** — and the deleted test's docstring spelled out the failure mode in detail.

The pre-fix code was deliberately sending `encoding_format=None` to the OpenAI SDK to *suppress* the SDK's `base64` default, because non-OpenAI providers reachable through the `openai/` path (notably Gemini, per the deleted comment) reject or misbehave on `encoding_format=base64`. The new code sends `encoding_format="float"` instead. For OpenAI itself this is fine — `"float"` is a valid encoding. But for the cross-provider case (a caller using `openai/...` model name with a custom `api_base` pointing at e.g. a Gemini-compatible endpoint or a vLLM server that doesn't implement encoding_format), the new code now sends `encoding_format="float"` where it previously sent `None`. If the downstream provider doesn't recognize `encoding_format` *at all*, sending `"float"` is no better (and possibly worse, depending on how strict the provider's parameter validation is) than sending `"base64"`.

**Concrete questions that need answers before this can land:**

1. **Is the deleted test's premise (Gemini rejects `encoding_format=base64`) still true today?** If yes, it's almost certainly true that Gemini also rejects `encoding_format=float`, in which case this PR breaks Gemini-via-OpenAI-shim embedding calls that currently work.
2. **What's the actual user-reported issue this PR is fixing?** The PR body says "OpenAI-compatible embedding calls now resolve `encoding_format` when unset instead of passing `None` to the SDK", but doesn't link a bug report or cite a provider that rejects `encoding_format=None`. The deleted comment in `main.py:4926` ("Omiting causes openai sdk to add default value of 'float'") *contradicts* the deleted test's docstring (which says the SDK default is `'base64'`) — one of the two is wrong, and the PR doesn't establish which.
3. **What's the migration path for callers who were depending on the `None`-suppression behavior?** A caller who's currently relying on `litellm` to send `None` to suppress the SDK default for a Gemini endpoint would silently start sending `"float"` after this merge. There's no opt-out (the chain has no `"unset"` sentinel that would restore the prior `None` behavior).

## Suggested resolution

Either:

- **(a) Keep the existing `None`-passthrough as the default** and *layer* the env-var override on top: `if encoding_format is None and not env_var_set: optional_params["encoding_format"] = None`. This preserves the prior cross-provider compatibility while adding the env-var control surface.
- **(b) Establish via test or PR-body cite that the `None`-suppression behavior is no longer needed** (e.g. confirm with current Gemini-via-OpenAI-shim that `encoding_format="float"` is now accepted, OR demonstrate that the `None` path was never actually suppressing anything because the OpenAI SDK serializes `None` as omitted-field on the wire anyway). Then this PR is fine as-is, with a CHANGELOG note flagging the behavior change.
- **(c) Add a sentinel** like `LITELLM_DEFAULT_EMBEDDING_ENCODING_FORMAT="omit"` (or `""` after the `or` chain) that restores the prior pass-`None` behavior for the cross-provider escape hatch. This costs one extra arm in the resolution chain but lets users opt out without code changes.

## Other nits (independent of the main concern)

- **The env-var lookup at `:4926` runs on every `embedding()` call**, not just when `encoding_format is None`. If `get_secret_str` is anything more expensive than `os.environ.get`, this is a hot-path regression. Worth a `if encoding_format is None: ...lookup...` guard inside the `else` branch (already there structurally but worth a comment so future contributors don't move the lookup out).
- **The chain uses Python `or`, which short-circuits on falsy values** — so `LITELLM_DEFAULT_EMBEDDING_ENCODING_FORMAT=""` would silently fall through to `"float"`. Probably fine but worth a one-line note in the README that empty-string env values are treated as unset.
- **No regression test for the `optional_params.get("encoding_format")` second tier of the chain.** The PR adds tests for explicit-arg (tier 1), env-var (tier 3), and default (tier 4), but not for the case where some upstream code path already populated `optional_params["encoding_format"]` and the resolution should pick that up.

## Verdict

**needs-discussion.** The mechanics of the fix are clean (correct precedence chain, correct env-var primitive, good test coverage of the new chain, good docs). But the PR removes a deliberately-placed `None`-suppression with a docstring explaining its purpose, without establishing whether the cross-provider compatibility that was being preserved is still relevant. Need either confirmation that the `None`-suppression is no longer load-bearing, OR an opt-out sentinel in the new resolution chain, OR keep `None` as the default and layer the env-var on top. The PR-body lacks the "we verified this against the providers the deleted test mentioned" evidence that would make this a confident `merge-after-nits`.
