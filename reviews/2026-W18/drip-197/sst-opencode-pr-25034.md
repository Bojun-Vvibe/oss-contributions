# sst/opencode PR #25034 — feat: default HTTP API backend to on for dev/beta channels

- PR: https://github.com/sst/opencode/pull/25034
- Head SHA: `1e5cc6da19c316ba2f722c26b3ebba9f027d9890`
- Files touched: 1 (`packages/core/src/flag/flag.ts` +13/-1).

## Specific citations

- New module-level constant `HTTPAPI_DEFAULT_ON_CHANNELS = new Set(["dev", "beta", "local"])` at `packages/core/src/flag/flag.ts:14-16`, with explanatory comment that "stable (`prod`/`latest`) installs" stay on the legacy hono backend.
- The behavioral flip is at `packages/core/src/flag/flag.ts:88-94`: the previous one-line `OPENCODE_EXPERIMENTAL_HTTPAPI: truthy("OPENCODE_EXPERIMENTAL_HTTPAPI")` becomes a three-clause expression — `truthy(...)` OR `(!falsy(...) && HTTPAPI_DEFAULT_ON_CHANNELS.has(InstallationChannel))`. Result: explicit `true/1` always wins, explicit `false/0` always wins (escape hatch), and unset → channel-derived default.
- New import at `:2`: `import { InstallationChannel } from "../installation/version"`. The InstallationChannel symbol is presumably resolved at module-load time, which makes this branch evaluated once per process — fine for a long-lived agent process, but see nit #2.

## Verdict: merge-after-nits

## Concerns / nits

1. **`local` channel default-on is a wider surface than the PR title implies.** PR title says "dev/beta channels"; the constant at `:16` actually includes `"local"` — i.e., source-tree builds run by contributors. That's the correct call (contributors should dogfood the new backend) but the title and PR body should mention `local` explicitly. Otherwise a contributor doing a clean source build will silently flip backends and may attribute regressions to other changes.
2. **`InstallationChannel` is imported as a value, not a function** (`HTTPAPI_DEFAULT_ON_CHANNELS.has(InstallationChannel)`). If the channel is computed lazily from a stat or an env var read at import time, that's fine; if it's reactive (e.g., re-reads a file), then the `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI` value is captured at the *first* evaluation. The surrounding comment block at `:96-98` ("Evaluated at access time (not module load) because tests, the CLI, and …") suggests that comment may apply to the *next* flag, not this one. Worth a one-line confirmation that `InstallationChannel` is stable post-bootstrap, or wrap this expression in a getter so it's recomputed.
3. **No test added.** This is a behavioral default change for two install channels. A unit test that mocks `InstallationChannel = "dev"` and asserts `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI === true` (and the same for `prod` returning `false`, plus the explicit env-var override matrix) would lock the policy. The flag module is exactly the right surface to test directly because it's pure logic over `process.env` + a constant.
4. **`falsy` returns `true` only for `"false"`/`"0"`** (per `:11-13`). So an explicit `OPENCODE_EXPERIMENTAL_HTTPAPI=no` does *not* count as falsy and therefore falls through to the channel default. That's consistent with how the rest of the file handles env vars but worth documenting the exact accepted strings for the escape-hatch in the inline comment.
