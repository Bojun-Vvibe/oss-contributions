# sst/opencode#24363 — fix(agent): accept common color names
**SHA**: `e955e50d` · **Author**: pascalandr · **Size**: +70/-6 across 5 files

## What
Extends the agent `color` config to accept twelve common named colors
(`black`, `white`, `red`, `orange`, `yellow`, `green`, `blue`, `purple`,
`pink`, `cyan`, `gray`, `grey`) in addition to existing hex strings and
theme tokens. The TUI render path adds a `namedAgentColors` lookup that
maps each name to a fixed hex (Tailwind 500-ish palette), with a
fallback chain to the theme record and finally the rotating index palette.
Schema, generated SDK types, and docs are kept in sync.

## Specific cite
- `packages/opencode/src/cli/cmd/tui/context/local.tsx:16-28` introduces
  the new `namedAgentColors: Record<string, string>` map. The fallback
  expression on line 110-111 is the meaningful safety net:
  `(theme[color as keyof typeof theme] as RGBA | undefined) ?? colors()[index % colors().length]`
  — previously this site cast unconditionally and would have produced
  `undefined` at runtime for any unknown string that slipped past the
  schema. That cast removal is the real bugfix; the named-color list is
  the headline feature.
- `packages/opencode/src/config/agent.ts:21-34` adds the literal union
  to the `Color` schema so the input is validated server-side, not just
  in the renderer.
- `packages/sdk/js/src/v2/gen/types.gen.ts:1255-1325` is a generated
  file but is checked in — the diff confirms regeneration was run.
- Test `packages/opencode/test/config/agent-color.test.ts:25,37` covers
  the `"blue"` case end-to-end through `load()`.

## Verdict: merge-as-is
Small, well-scoped, schema/runtime/SDK/docs all coherent, with a test.
The implicit fallback hardening at `local.tsx:111` is a quiet bonus —
it eliminates a class of "valid-by-config, undefined-at-runtime" bugs
without expanding scope.
