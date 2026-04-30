# QwenLM/qwen-code #3753 — fix(cli): honor proxy setting

- **URL:** https://github.com/QwenLM/qwen-code/pull/3753
- **Head SHA:** `5bebe5b78289de03d318f0990d2352e241be816c`
- **Files:** `docs/users/configuration/settings.md` (+8/-1), `packages/cli/src/config/config.test.ts` (+33/-0), `packages/cli/src/config/config.ts` (+8/-6 — net +2 with reformatting), `packages/cli/src/config/settingsSchema.test.ts` (+12/-0), `packages/cli/src/config/settingsSchema.ts` (+11/-0), `packages/vscode-ide-companion/schemas/settings.schema.json` (+4/-0)
- **Verdict:** `merge-after-nits`

## What changed

Adds a top-level `proxy` setting that fits between `--proxy` CLI flag and the `HTTPS_PROXY`/`HTTP_PROXY` env-var fallbacks. Three layers:

1. **Schema definition** at `packages/cli/src/config/settingsSchema.ts:290-300`:
   ```typescript
   proxy: {
     type: 'string',
     label: 'Proxy',
     category: 'Advanced',
     requiresRestart: true,
     default: undefined as string | undefined,
     description: 'Proxy URL for CLI HTTP requests. Takes precedence over proxy environment variables when --proxy is not provided.',
     showInDialog: false,
   },
   ```
   Sits as a top-level key (not nested) at the same depth as `general`/`ui`/`advanced`. The `requiresRestart: true` is correct — proxy resolution is consumed at process boot, mid-session changes can't take effect.

2. **Resolution chain** at `packages/cli/src/config/config.ts:1245-1253`:
   ```typescript
   proxy:
     argv.proxy ||
     settings.proxy ||
     process.env['HTTPS_PROXY'] ||
     process.env['https_proxy'] ||
     process.env['HTTP_PROXY'] ||
     process.env['http_proxy'],
   ```
   Strict 6-tier precedence: `--proxy` flag > settings.json `proxy` > env vars in upper-then-lower-cased pairs.

3. **Documentation** at `docs/users/configuration/settings.md:74-82` adds a new "top-level" section above the per-category tables:
   > Settings are organized into categories. Most settings should be placed within their corresponding top-level category object in your `settings.json` file. A few compatibility settings, such as `proxy`, are top-level keys.
   
   Plus an example block at `:478` showing `"proxy": "http://localhost:7890"` at the root.

4. **Test coverage** is genuinely thorough for the resolution-chain at `config.test.ts:794-832`:
   - Settings-only proxy → wins (`:794-800`).
   - Schemeless `localhost:7890` → normalized to `http://localhost:7890` (`:802-808`). This is the load-bearing test for an existing normalizer somewhere in `loadCliConfig` that prepends `http://` if missing.
   - Settings proxy beats `HTTPS_PROXY` env (`:810-817`).
   - CLI `--proxy` beats settings proxy (`:826-832`).
   
   And schema-shape coverage at `settingsSchema.test.ts:122-130` asserts schema metadata (`type === 'string'`, `category === 'Advanced'`, `requiresRestart === true`, `default === undefined`, `showInDialog === false`).

5. **VSCode companion JSON schema** at `packages/vscode-ide-companion/schemas/settings.schema.json:36-39` adds the `proxy` property so settings.json autocomplete in VS Code surfaces it.

## Why it's right

- **Real ergonomic gap.** Without this, users had to either pass `--proxy` on every invocation (painful for CI scripts) or set `HTTPS_PROXY` in their shell (leaks to every other tool, often unwanted in container environments). A persistent setting at the user/project settings.json level is the natural place.
- **Precedence chain is honest.** CLI flag wins (explicit-now intent), then settings (persistent declaration), then env (ambient inherit). The 4-tier env fallback covers all four common conventions (`HTTPS_PROXY` / `https_proxy` / `HTTP_PROXY` / `http_proxy`) which is correct for a CLI that may speak both HTTPS and HTTP backends.
- **Schemeless normalization** at `:802-808` is a quality-of-life detail that prevents the most common config error (forgetting `http://`). The actual normalizer isn't visible in the diff slice — it's elsewhere in `loadCliConfig` — but the regression test locks the contract.
- **`showInDialog: false`** at `settingsSchema.ts:298` is the right call. A proxy URL is the kind of thing users set once at install time and forget, not something they'd toggle from a runtime settings dialog. Keeping it out of the dialog reduces noise without restricting access via direct settings.json edit.
- **`requiresRestart: true`** at `:294` is honest — proxy is consumed at process boot, so a hot-reload would silently no-op. The flag at least surfaces this constraint in the settings UI.
- **Schema test ordering at `:114-121`** asserts the new `proxy` key falls between `telemetry` and `model` in the canonical schema-keys list. This locks insertion ordering, which matters for documentation generators that walk the schema in declaration order.

## Nits / blockers

1. **`||` vs `??` for the resolution chain.** `argv.proxy || settings.proxy || ...` at `config.ts:1247-1253` will fall through to env vars if `settings.proxy` is the empty string `""`. That's *probably* fine because `""` is a meaningless proxy URL, but a user who explicitly wants to *disable* a previously-inherited env-var proxy by setting `"proxy": ""` in settings.json instead gets the env-var picked up. A `??` chain would respect the empty-string-as-disable convention. Worth picking one and documenting which.

2. **No test for the disable-via-empty-string case.** Related to #1: if the answer is "empty string falls through to env" (the current `||` shape), document that explicitly. If it's "empty string disables proxy", switch to `??` and add a test asserting `settings = {proxy: ""}` plus `HTTPS_PROXY=http://env` → `getProxy() === undefined`.

3. **Schemeless normalization location is invisible.** The test at `:802-808` asserts `settings.proxy = 'localhost:7890'` becomes `getProxy() === 'http://localhost:7890'`, but the actual `http://`-prepending logic isn't in this PR's diff slice. Worth confirming it lives in a normalizer that runs *after* the resolution-chain at `:1247-1253`, otherwise the `||` chain might bail out early on a schemeless `argv.proxy = "localhost:7890"` before normalization can run. (Probably fine — the test would fail if it weren't — but the diff slice doesn't show the wiring.)

4. **No upper-cased / lower-cased env-var test for the new path.** The PR adds tests for `HTTPS_PROXY` (line `:812`) but not for `https_proxy`/`HTTP_PROXY`/`http_proxy` interaction with the new `settings.proxy` tier. The four env-var checks at `:1248-1252` aren't exercised independently. Worth one parametrized test that asserts settings proxy beats *each* of the four env-var conventions.

5. **`packages/vscode-ide-companion/schemas/settings.schema.json:36-39`** adds `"proxy"` at the top level alongside `mcpServers`. Confirm this file is the authoritative schema for VSCode autocomplete; if there's also a generated schema (some monorepos keep both), this PR should update both, not just the source. The diff slice shows only one file changing.

6. **`settings.md` example at `:478`** places `"proxy"` as the first key in the example settings.json. Convention in most docs is to keep canonical examples grouped by category (general, ui, etc.) — putting a single top-level key first reads slightly oddly. Cosmetic but the doc is the user-facing surface.

## Risk

Low. Pure additive feature — existing users without `proxy` in settings.json see zero change. The resolution-chain order at `:1247-1253` preserves the existing CLI > env behavior and slots the new tier at the only sensible point (between explicit CLI and ambient env). The `requiresRestart: true` flag honestly surfaces the boot-time-only constraint. The biggest *operational* risk is users setting a wrong proxy URL in committed project settings.json and breaking their teammates' connectivity — but that's a UX risk inherent to any persistent proxy config, not specific to this PR. The empty-string-as-disable ambiguity (#1) is the only behavior question worth resolving before merge.
