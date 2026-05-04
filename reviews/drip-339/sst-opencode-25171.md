# Review: sst/opencode #25171

- **Title:** feat(tui): add content_padding option to control horizontal content margins
- **Head SHA:** `993118cf84f1eecb0b0a09a908672967c60fe465`
- **Scope:** +10 / -2, two files (`tui-schema.ts`, `routes/session/index.tsx`)
- **Drip:** drip-339

## What changed

Adds a new optional `content_padding` numeric field to the TUI schema and
threads it through the session route to override the default horizontal
margin around the message stream.

## Specific observations

- `packages/opencode/src/cli/cmd/tui/config/tui-schema.ts` (+7): new
  `content_padding: z.number().int().min(0).max(...).optional()` entry.
  Confirm the upper bound is reasonable for narrow terminals (≤ ~10) so a
  user typo of `100` doesn't push the entire stream off-screen.
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx` line ~lhs of
  the diff (the 2 deletions): the existing hard-coded padding constant is
  replaced with `config.content_padding ?? <default>`. Make sure the
  `??` (not `||`) is used so an explicit `0` is honored.
- No documentation update for the new schema field; users will only see it
  in the JSON-schema autocomplete. A one-liner under the TUI config docs
  would help discoverability — nit.
- No test coverage for the new option. Given it's a passthrough numeric
  prop, a snapshot of the rendered route at `content_padding: 0` and a
  large value would catch regressions cheaply.
- The schema field name `content_padding` is consistent with neighbouring
  snake_case keys in `tui-schema.ts`; good.

## Risks

- Unbounded upper value could break layout; cap it.
- Otherwise low-risk additive change with a sane default fallback.

## Verdict

**merge-after-nits** — cap the schema max, confirm `??` semantics, and add
a one-line doc entry.
