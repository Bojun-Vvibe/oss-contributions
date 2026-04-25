# cline/cline #10397 — feat: add API key field to LM Studio provider

- **Repo**: cline/cline
- **PR**: [#10397](https://github.com/cline/cline/pull/10397)
- **Head SHA**: `a20608766bf7a13883c907cb201d96f5615572b5`
- **Author**: pierluigilenoci
- **State**: OPEN (+30 / -5)
- **Verdict**: `merge-after-nits`

## Context

LM Studio's server has a `Require Authentication` toggle. When
enabled, Cline can't connect because the LM Studio handler hard-
codes `apiKey: "noop"`. Workaround was to switch to the "OpenAI
Compatible" provider, which loses LM Studio-specific niceties
(model discovery, context window probing). This adds an optional
`lmStudioApiKey` field plumbed through the same paths as every
other provider key.

## Design

Mechanical end-to-end addition of one secret. Eight files, all
the touch points you'd expect from a Cline provider key:

1. **Proto** (`proto/cline/models.proto:202`): adds
   `optional string lm_studio_api_key = 40;` to
   `ModelsApiSecrets`, and field 88 to `ModelsApiConfiguration`
   (line 604). Field numbers `40` and `88` are next-available — no
   collision risk.
2. **Secrets storage** (`src/shared/storage/state-keys.ts:313`):
   adds `"lmStudioApiKey"` to `SECRETS_KEYS` so the value lands
   in the secrets backend, not plaintext settings.
3. **Provider key map** (`src/shared/storage/provider-keys.ts:59`):
   `lmstudio: "lmStudioApiKey"` lets the generic
   `getKeyForProvider(provider)` lookup find it.
4. **Handler** (`src/core/api/providers/lmstudio.ts:33`):
   `apiKey: this.options.lmStudioApiKey || "noop"`. The `"noop"`
   fallback preserves today's behavior for unauthenticated LM
   Studio servers — important, that's the common case.
5. **Model probe** (`src/core/controller/models/getLmStudioModels.
   ts:12-22`): the previously-unused `_controller` is renamed to
   `controller` (line 12) so the model list endpoint can also
   read the API key from `controller.stateManager.
   getApiConfiguration()` and send `Authorization: Bearer <key>`
   via the `headers` object. Without this, "auto-discover models"
   would 401 even after the user added a key.
6. **Proto conversion** (`api-configuration-conversion.ts:463,
   644`): adds the field to both `convertApiConfigurationToProto`
   and `convertProtoToApiConfiguration`. Symmetric — round-trip
   safe.
7. **UI** (`webview-ui/.../LMStudioProvider.tsx:104-110`): drops
   in the standard `<ApiKeyField>` with sensible help text and
   `(optional)` placeholder.
8. **Configured-providers heuristic** (`getConfiguredProviders.ts:
   202-204`): adds `apiConfiguration.lmStudioApiKey` to the OR-
   chain so a user who only set the API key (no custom base URL,
   no model yet) still shows up as having configured LM Studio.
   Small UX detail, easy to miss.

## Risks

- The `apiKey: this.options.lmStudioApiKey || "noop"` (line 33)
  treats empty string the same as undefined, which is correct —
  `<ApiKeyField>` empties to `""` not `null` when the user clears
  it.
- `Authorization` header is only added when the key is truthy
  (`getLmStudioModels.ts:18-21`). So a previously-anonymous LM
  Studio server still works without sending a `Bearer` header.
  Good.
- No new test coverage. For an 8-file plumbing add this is
  forgivable, but a single integration test exercising "auth
  enabled → models endpoint succeeds" would close the loop on
  the `getLmStudioModels` change which is the real correctness
  risk (a typo there silently breaks model discovery).

## Suggestions

- Add one test: `getLmStudioModels` sends `Authorization: Bearer
  <key>` when configured, and no header when not.
- The proto comments on field 40 / 88 don't say what the field is.
  Other entries in this proto (e.g. lines 200-201) also skip
  comments so it's consistent — but for a secret field it's
  worth a `// LM Studio "Require Authentication" key` so future
  rotators don't have to grep.

## What I learned

Cline's provider-key plumbing has 8 mandatory touch points
(proto secret → state-keys SECRETS list → provider-keys map →
handler → controller probe → proto conversion both directions →
UI field → configured-providers heuristic). Missing any of them
half-breaks the feature in a different way: skip SECRETS_KEYS
and the value persists in plaintext; skip the heuristic and the
provider doesn't show as configured; skip the controller probe
and discovery 401s after auth-enable. This PR hits all eight,
which is the right shape and a good reference for future provider
additions.
