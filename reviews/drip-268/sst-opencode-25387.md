# sst/opencode #25387 — feat(tui): configurable custom home placeholders

- **Head SHA:** `0787159ed973370e9805ffca2aeae67571567ab7`
- **Files:** `bun.lock`, `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx`, `packages/opencode/src/cli/cmd/tui/config/tui-schema.ts`, `packages/opencode/src/cli/cmd/tui/routes/home.tsx` (+35/-5)
- **Verdict:** `merge-after-nits`

## Rationale

User-facing knob to override the rotating "Ask anything... \"...\"" / "Run a command... \"...\"" placeholder text on the home screen with literal user-supplied strings, configured via `tui.placeholders.{input,shell}` in the zod schema (`tui-schema.ts:33+`). The wiring is clean: `PromptProps.placeholders` gets two new optional `rawNormal`/`rawShell` booleans (`prompt/index.tsx:60`), and the placeholder memo at `prompt/index.tsx:1018` branches on them — when raw, it returns the example string verbatim instead of wrapping with the prefix. `home.tsx` reads `useTuiConfig()` and threads the user's arrays plus `rawNormal: true`/`rawShell: true` down. The `bun.lock` churn is incidental noise from `@solidjs/start` and `ghostty-web` losing their integrity hashes — unrelated to this feature.

Nits: (1) the schema names `input`/`shell` but the prop names are `normal`/`shell` and the new flags are `rawNormal`/`rawShell` — three names for the same concept will bite a future contributor; pick one (`input` everywhere is most user-facing); (2) when both default and user arrays are configured, the rotation index `store.placeholder % list().length` advances against the user list — fine, but document that built-in examples are *replaced* not *appended* (the schema description says "Replaces" so this is consistent; just make sure release notes match); (3) the `bun.lock` integrity-hash drops should be reverted — that looks like a stale local lockfile, not an intentional change for this PR.

Low risk: the feature is opt-in via config, the default code path is byte-identical, and there's no validation cost on the hot render path beyond a single `?:`.
