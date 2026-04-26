# Review — QwenLM/qwen-code#3624: fix(cli): add API Key option to `qwen auth` interactive menu

- **Repo:** QwenLM/qwen-code
- **PR:** [#3624](https://github.com/QwenLM/qwen-code/pull/3624)
- **Author:** doudouOUC (jinye)
- **Head SHA:** `0528b02d7f85f49b30af41f3489fdf5036009775`
- **Size:** +466 / −111 across 5 files (3 docs files +9/−2 total, `auth.ts` +11/−1, `auth/handler.ts` +446/−108)
- **Verdict:** `merge-after-nits`

## Summary

Adds a new `qwen auth api-key` standalone subcommand and an "API Key — Bring your own API key" entry in the interactive menu shown by `qwen auth`. Updates three docs files (`docs/users/configuration/auth.md`, `docs/users/features/commands.md`, `docs/users/quickstart.md`) to reflect the new option, reorders the menu so Coding Plan is shown first and the discontinued Qwen OAuth last, and rewrites large chunks of `packages/cli/src/commands/auth/handler.ts` (+446/−108) to introduce `handleApiKeyAuth()` alongside the existing `handleQwenAuth` and `runInteractiveAuth`.

## Technical assessment

The user-facing change is straightforward and aligns with the existing CLI surface: `qwen auth qwen-oauth` (legacy), `qwen auth coding-plan` (preferred), `qwen auth coding-plan --region china --key sk-sp-…` (scriptable), and now `qwen auth api-key` (BYO key). The `apiKeyCommand` block at `packages/cli/src/commands/auth.ts:54-60` mirrors the existing `codePlanCommand` and `qwenOauthCommand` shape and registers via `.command(apiKeyCommand)` in the yargs builder at the visible bottom of the diff.

The docs updates are clean. `docs/users/configuration/auth.md:333` adds "API Key — Bring your own API key" between Coding Plan and the discontinued OAuth, which matches the menu order in `runInteractiveAuth`. The command-table update at `auth.md:344` and `commands.md:243` is mechanical and consistent. `quickstart.md:226` adds the one-line `qwen auth api-key` row.

The risky part is the `+446/−108` in `auth/handler.ts`. Net +338 in a handler that already had three auth flows is large. Without seeing the full file diff in this review window, the structural concern is whether `runInteractiveAuth` still hands off cleanly when the user picks the API-key option — i.e. whether the new `handleApiKeyAuth()` is implemented in-place or whether it's a third orchestration path with its own prompt loop, error handling, and storage write. Either is fine, but if it duplicates prompt-loop logic from the Coding Plan handler, that should be extracted.

## Nits worth addressing pre-merge

1. **Handler diff is large and not easily reviewable from gh pr diff.** +446/−108 in one file is the kind of change where a side-by-side review of the new `handleApiKeyAuth` against the existing `handleQwenAuth` is mandatory. Specifically, look for: (a) duplicated input-validation regex (API-key shape varies — `sk-` for API-key path vs `sk-sp-` for Coding Plan), (b) duplicated keychain/file-write logic, (c) consistent error messages (don't drift from "Invalid key format" vs "Bad API key" between handlers).

2. **`Authenticate using an API key` is undertyped.** At `auth.ts:56` the `describe: t('Authenticate using an API key')` doesn't say *which* API endpoint the key is for. Coding Plan uses `sk-sp-…` against DashScope; "API Key" here presumably means a generic OpenAI-compatible base URL. The interactive handler should prompt for `base_url` AND `api_key`, not just the key. If it doesn't, the user has to also set `OPENAI_BASE_URL` env or edit settings JSON — defeating the purpose of an interactive flow.

3. **Menu reordering is a UX change.** Previous menu order put Coding Plan first and OAuth (still functional at the time) second. New order is Coding Plan → API Key → discontinued OAuth. That's correct given OAuth's status, but worth a brief CHANGELOG note since users who muscle-memory'd "second option = OAuth" will now hit "API Key" by accident the first time.

4. **Doc table inconsistency.** `docs/users/configuration/auth.md:344` lists `qwen auth api-key` without a `--key` shortcut. `docs/users/features/commands.md:243` mirrors that. If `qwen auth api-key --key sk-…` is supported (matching the scriptable `coding-plan --region china --key sk-sp-…` pattern), document it. If not, consider adding it for symmetry — a CI/scripting use case that the Coding Plan path already supports.

5. **`Configure Qwen authentication with Coding Plan, API Key, or Qwen-OAuth`.** The describe string at `auth.ts:73` lists Qwen-OAuth without the discontinued caveat. Either add `(legacy)` or drop OAuth from the top-level describe since the menu and commands.md table already mark it discontinued.

## Verdict rationale

`merge-after-nits`. The user need is real (BYO-key users were previously forced to edit settings JSON), the docs are updated consistently, and the menu reorder is sensible. The +446/−108 handler diff needs a careful in-PR side-by-side review and the `--key` non-interactive shortcut should be either added or explicitly declined for symmetry with the Coding Plan path. None of the nits require a rewrite.
