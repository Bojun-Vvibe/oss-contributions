# charmbracelet/crush#2674 — APIURL/APIKEY env overrides with CRUSH_PROVIDER targeting

**What changed.** Adds env-var overrides for provider endpoint and API key, applied during `configureProviders`:
- Provider-specific: `CRUSH_<PROVIDER>_BASE_URL`, `<PROVIDER>_BASE_URL`, `CRUSH_<PROVIDER>_API_KEY`.
- Generic: `CRUSH_API_URL`/`API_URL`/`APIURL`, `CRUSH_API_KEY`/`API_KEY`/`APIKEY`. These apply to the provider named by `CRUSH_PROVIDER` (default `openai`).
- Provider-ID normalization (`providerEnvSuffix`) uppercases letters/digits and collapses other runes to `_`, then trims trailing underscores.

Three new tests cover OpenAI base-URL override, generic vars hitting default OpenAI, and generic vars targeting a custom provider via `CRUSH_PROVIDER`.

**Why it matters.** Lets users point Crush at a corporate gateway / local proxy without editing config files — a common request when one user juggles multiple deployments.

**Concerns.**
1. **Generic `APIKEY` / `APIURL` are dangerously common names.** A bare `API_KEY` or `APIKEY` env var is set by countless other tools (CI runners, IDE plugins, prior `export` history) and almost never intended for Crush. With `CRUSH_PROVIDER` defaulting to `openai`, a user who has `APIKEY=<some-other-service-token>` exported will silently send that token to OpenAI on the next Crush invocation. **This is a credential-leak vector.** Recommend either (a) requiring `CRUSH_PROVIDER` to be set explicitly before honoring generic `APIKEY`/`APIURL`, or (b) dropping the unprefixed `APIKEY`/`APIURL`/`API_KEY`/`API_URL` aliases entirely and only honoring `CRUSH_API_KEY`/`CRUSH_API_URL`.
2. **Two override sites, easy to drift.** `configureProviders` calls `providerEnvOverrides` once for catwalk-known providers (line ~211) and once for user-defined providers (line ~342). Any future field added to provider config (headers? extra params?) needs both sites touched. Worth pulling the apply step into a single helper.
3. **`firstEnvValue` does `TrimSpace` per key but the final return also `TrimSpace`s** — minor double-trim, not a bug.
4. **No precedence test against existing config-file values.** Tests verify env overrides take effect; they don't verify the env override beats an explicit value already set in `crush.json`. Document the intended precedence in the README.
5. **`providerEnvSuffix` strips Unicode marks unpredictably.** A provider ID like `mistral-fr` becomes `MISTRAL_FR`, but `mistral.fr` also becomes `MISTRAL_FR` — collision risk for adjacent IDs. Probably theoretical.

Land only after (1) is addressed; the leak risk is real.
