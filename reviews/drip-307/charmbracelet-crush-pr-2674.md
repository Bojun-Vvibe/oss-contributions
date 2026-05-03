# Review: charmbracelet/crush #2674 — config: support APIURL/APIKEY env overrides with CRUSH_PROVIDER targeting

- **Repo**: charmbracelet/crush
- **PR**: #2674
- **Head SHA**: `038b447dd9f2dff8dd10b49534685124d6767fe2`
- **Author**: axeprpr

## What it does

Adds a multi-tier env-var override system for provider endpoint /
API key, so users can repoint Crush at a custom gateway without
editing `crush.json`. Three tiers, in priority order:

1. Provider-specific: `CRUSH_<PROVIDER>_BASE_URL`,
   `<PROVIDER>_BASE_URL`, `CRUSH_<PROVIDER>_API_KEY`.
2. Generic, gated by `CRUSH_PROVIDER=<id>` (defaults to `openai`):
   `CRUSH_API_URL` / `API_URL` / `APIURL`, and the `_KEY` mirrors.
3. Existing `$OPENAI_API_KEY` template substitution preserved.

## Diff notes

- `internal/config/load.go:212-220` — first override site, applied to
  catwalk `knownProviders` before the per-provider config block is
  built. Calls `providerEnvOverrides(env, string(p.ID))`. Mutates the
  `p` copy in place, then the rest of the loop reads from it.
- `internal/config/load.go:342-350` — second override site, applied to
  user-defined providers (the `for id, providerConfig := range ...`
  branch). Same helper, different struct field names (`BaseURL` vs
  `APIEndpoint`). The duplication is necessary because the two
  branches operate on different struct types — but it does mean a
  future field addition has to be remembered in two places.
- `internal/config/load.go:410-440` — `providerEnvOverrides` does the
  three-tier resolution, with generic fallback gated by
  `CRUSH_PROVIDER` matching the resolving provider's id. Defaulting
  `target` to `"openai"` when unset is the right behavior — preserves
  the existing `OPENAI_BASE_URL` convention while adding the
  `CRUSH_PROVIDER` opt-in for non-openai users.
- `internal/config/load.go:442-460` — `providerEnvSuffix` upper-cases
  and replaces non-alphanumerics with `_`, collapsing runs and
  trimming. So `provider-id` → `PROVIDER_ID`. Reasonable, but
  `unicode.IsLetter` accepts non-ASCII letters (e.g. `é` → `É`),
  which then get embedded in env-var names — most shells reject that.
  Worth restricting to ASCII letters explicitly.
- `internal/config/load_test.go:136-225` — three new tests cover the
  three documented scenarios. Tests are well-isolated and don't share
  global state. Missing: a test that asserts the **provider-specific**
  override (`CRUSH_OPENAI_BASE_URL`) takes precedence over the
  generic (`API_URL` with `CRUSH_PROVIDER=openai`). The current code
  does the right thing via `cmp.Or`, but a regression test would
  pin it.

## Concerns

1. **Bare env-var names `APIURL` / `APIKEY` are dangerous.** They
   collide with whatever else the user might have in their environment
   for unrelated tools. Even gated by `CRUSH_PROVIDER`, the generic
   names are too generic. Recommend dropping `APIURL` / `APIKEY` and
   keeping only `API_URL` / `API_KEY` and the `CRUSH_`-prefixed
   variants. The `CRUSH_*` ones are the safe canonical form.
2. The default-to-`openai` behavior when `CRUSH_PROVIDER` is unset
   means a user who exports `API_URL` for their `curl`-ing scripts
   will silently override Crush's openai endpoint. That's a footgun.
   At minimum, log a warning when the generic-name fallback fires.
3. Two override-application sites (lines 212 and 342) copy-paste the
   same six-line if/if/if pattern. Extract to a small helper that
   takes pointers / setter funcs so the next field addition (e.g.
   `OrganizationID`) doesn't drift between branches.
4. ASCII-only restriction in `providerEnvSuffix` — see diff notes.

## Verdict

request-changes
