# google-gemini/gemini-cli#26445 — feat: add ignoreLocalEnv setting and --ignore-env flag

- PR ref: `google-gemini/gemini-cli#26445`
- Head SHA: `c0890e743e1853667f9fb38e98f793f31dd7713d`
- Title: feat: add ignoreLocalEnv setting and --ignore-env flag (#2493)
- Verdict: **merge-as-is**

## Review

Tightly scoped feature with strong test coverage. The semantics are exactly right:
`advanced.ignoreLocalEnv` (added in `packages/cli/src/config/settingsSchema.ts:+10`
and `schemas/settings.schema.json:+7`) skips the project-tree `.env` walk *but*
preserves `.gemini/.env` and the home-dir `.env` — verified by the dedicated test
case at `packages/cli/src/config/settings-env-isolation.test.ts:91-106` ("should
still load .gemini/.env even if ignoreLocalEnv is true"). That's the correct
distinction: project `.env` files are the credential-leak risk in untrusted
workspaces; the `.gemini/` namespaced one is intentional Gemini configuration.

The `--ignore-env` CLI flag is wired with strict precedence over the setting: the
test at `:215-230` confirms the flag wins even when `ignoreLocalEnv: false` is
explicitly set, which matches the standard "flag overrides config" Unix convention.
The walk-up-to-home behaviour is also covered by the deep-directory test at
`:135-160` — local `.env` and all its parent `.env` files are skipped, but the home
`.env` is still picked up.

Particularly worth calling out: the "respect trust whitelist even when loading from
home `.env`" test at `:185-203` confirms that the existing trust-whitelist gate still
applies to the fall-through home env, so this feature doesn't accidentally widen the
attack surface for untrusted workspaces. Docs are updated in both
`docs/cli/settings.md:161` and `docs/reference/configuration.md:1755-1760`. Ship it.
