# block/goose #9015 — Local inference provider improvements

- **Head SHA:** `da10317a27e1077794bda898710366a2ccdad529`
- **Author:** @monroewilliams
- **Verdict:** `merge-after-nits`
- **Files:** +27 / −4 across `crates/goose/src/providers/declarative/{llama_swap,lmstudio,omlx}.json` (omlx.json is new) and `crates/goose/src/providers/provider_registry.rs`

## Rationale

Three tightly-related local-inference provider tweaks bundled cleanly. (1) New declarative provider `omlx.json` — modeled after the existing LM Studio entry but with port 8000 default and `OMLX_HOST` env var. (2) Two existing local providers (`llama_swap.json`, `lmstudio.json`) get their `api_key_env` field changed from `""` to `"LLAMA_SWAP_API_KEY"` / `"LMSTUDIO_API_KEY"` respectively. (3) The provider registry logic at `crates/goose/src/providers/provider_registry.rs:166-172` is corrected so that an API key field is exposed in the UI whenever `api_key_env` is non-empty, regardless of `requires_auth`.

The provider_registry fix is the substantive change and it's correct. Before the PR, the conditional was:

```rust
if config.requires_auth && !config.api_key_env.is_empty() {
    vec![ConfigKey::new(&config.api_key_env, true, true, None, true)]
}
```

which means a provider that supports *optional* API-key auth (e.g., LM Studio with the auth toggle enabled in the LM Studio UI, or a llama.cpp server behind nginx with a token) couldn't expose the field for the user to configure unless `requires_auth` was also `true` — but flipping `requires_auth` to true would then make goose *require* the key for users who don't have one configured, which breaks the no-auth default. The new conditional:

```rust
if !config.api_key_env.is_empty() {
    vec![ConfigKey::new(&config.api_key_env, config.requires_auth, true, None, true)]
}
```

decouples the two correctly: presence of `api_key_env` controls whether the UI surfaces the field, and `requires_auth` is passed through to `ConfigKey`'s second positional arg (the "required" flag) so the field is shown but optional. This matches the new omlx.json entry shape (`"requires_auth": false` + `"api_key_env": "OMLX_API_KEY"`) and keeps backward compatibility for providers that previously had `api_key_env: ""` (those still get an empty `config_keys` vector).

The new omlx.json (`crates/goose/src/providers/declarative/omlx.json:1-23`) follows the established declarative-provider shape: `engine: "openai"` (oMLX speaks the OpenAI-compatible API per the PR body), `display_name: "oMLX"`, single `OMLX_HOST` env var with `default: "http://localhost:8000"`, `dynamic_models: true`, `models: []`, `supports_streaming: true`, `requires_auth: false`, `skip_canonical_filtering: true`. The default port (8000 vs. LM Studio's typical 1234) is the correct distinction.

Three nits: (1) The PR uses positional args on `ConfigKey::new(&config.api_key_env, config.requires_auth, true, None, true)` — the second arg's meaning ("required") is now derived from `requires_auth` rather than hardcoded `true`, which is the substantive change, but a builder-style `ConfigKey::new(...).required(config.requires_auth)` would be much harder to misread; not blocking, but worth filing as cleanup. (2) The `api_key_env: "LLAMA_SWAP_API_KEY"` / `"LMSTUDIO_API_KEY"` env-var names are introduced fresh; the PR body doesn't say whether existing users running llama.cpp or LM Studio behind a token already have a conventional env var name they expect (e.g., upstream llama-server docs use `LLAMA_API_KEY`). Worth a quick check that the chosen names don't conflict with any existing user-facing convention. (3) `omlx.json:21` has `"requires_auth": false` but `"api_key_env": "OMLX_API_KEY"` — internally consistent with the registry fix in this PR, but if oMLX itself ships with auth-required as default in a future version, the JSON will need a flip; a comment or a follow-up issue would be useful.

Manual-testing-only is acceptable for declarative-provider JSON additions (the schema is validated at deserialize time by the existing test suite), and the `provider_registry.rs` change is small enough that the diff is the proof. Merge after a sanity-check on the chosen env var names.