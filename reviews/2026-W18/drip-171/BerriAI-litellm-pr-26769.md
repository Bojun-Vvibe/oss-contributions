# BerriAI/litellm#26769 — `feat: add FuturMix as named OpenAI-compatible provider`

- URL: https://github.com/BerriAI/litellm/pull/26769
- Head SHA: `6bd3935d3a60`
- Author: Gzhao-redpoint
- Size: +25 / -19 across 2 files

## Summary

Registers `futurmix` as a named OpenAI-compatible provider in
`litellm/llms/openai_like/providers.json`, with a corresponding
endpoint-support entry in `provider_endpoints_support.json`. The
provider declares chat/completions, messages, responses,
embeddings, and image generations as supported, with the standard
audio/moderations/batches/rerank disabled.

## Specific references

`litellm/llms/openai_like/providers.json` lines ~80–90 (after the
`gmi` entry):

```json
"futurmix": {
  "base_url": "https://futurmix.ai/v1",
  "api_key_env": "FUTURMIX_API_KEY",
  "api_base_env": "FUTURMIX_API_BASE",
  "param_mappings": {
    "max_completion_tokens": "max_tokens"
  }
}
```

`provider_endpoints_support.json`:

- New `futurmix` entry around line 1063 with the standard
  endpoint matrix.
- **Unrelated deletions** in the same file:
  - Removed `"a2a": false` and `"interactions": false` keys from
    an existing provider entry near lines 471–477 (visible at
    diff lines 31–34).
  - Removed the entire `charity_engine` provider entry at lines
    2553–2570 (visible at diff lines 70–84).

## Concerns

The two deletions noted above are unrelated to "add FuturMix" and
should not be in this PR. They look like merge-conflict resolution
or a stale local checkout. Without a justification in the PR body
they are silent regressions:

- Removing `charity_engine` entirely deletes a previously
  supported provider's discoverability metadata. If charity_engine
  was deprecated, that should be its own PR with a changelog
  entry.
- Removing the `a2a` and `interactions` flags from another
  provider changes the schema-shape for that provider only,
  causing schema heterogeneity across the file.

The FuturMix-only changes are otherwise straightforward and match
the structure of neighboring entries (e.g. `gmi`, `sarvam`).

## Risks

- The `param_mappings: { "max_completion_tokens": "max_tokens" }`
  is correct for an OpenAI-compatible upstream that uses the
  legacy `max_tokens` field. Sanity-check: confirm FuturMix's
  public docs actually require `max_tokens` not
  `max_completion_tokens`. If they accept both, the mapping is
  fine but redundant; if they only accept `max_completion_tokens`,
  this mapping would break the call.
- No model list / pricing entry is added in `model_prices_and_context_window.json`,
  so cost tracking will fall back to defaults. That's fine for a
  provider-only PR but worth flagging in the description.
- No tests, but provider-registration PRs in this repo
  conventionally don't add unit tests.

## Verdict

`request-changes`

Reason: the unrelated deletions in `provider_endpoints_support.json`
(charity_engine entry + `a2a`/`interactions` keys) need to be
reverted from this PR. Submit them separately if intentional.

## Suggested nits

- Confirm in the PR body that FuturMix's API expects `max_tokens`
  (not `max_completion_tokens`).
- Optional: add a docs page under `docs/my-website/docs/providers/futurmix.md`
  matching the URL declared in `provider_endpoints_support.json`.
