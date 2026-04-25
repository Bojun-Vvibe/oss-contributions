---
pr: 24308
repo: sst/opencode
sha: 31598e8ebf48d73933dab454b57be4d3b1f981ab
verdict: merge-after-nits
date: 2026-04-25
---

# sst/opencode#24308 — fix(config): preserve permission order with Effect decode

- **URL**: https://github.com/sst/opencode/pull/24308
- **Author**: kitlangton

## Summary

Migrates the remaining Zod-only schema islands inside `packages/opencode/
src/config/` to Effect Schema, and crucially routes config decoding
through `EffectSchema.decodeUnknownExit(schema)(data, { errors: "all",
propertyOrder: "original" })` so the order of permission keys in user
config is preserved end-to-end. This matters because permission rules
are first-match-wins, so reordering during validation changes runtime
authorization decisions.

## Reviewable points

- `packages/opencode/src/config/parse.ts:204` introduces the new
  `effectSchema()` entry point with `propertyOrder: "original"` — this
  is the load-bearing change. The function also enforces "no extra
  top-level keys" via `topLevelExtraKeys()` (line ~237), which mirrors
  Zod's `.strict()` behavior. Good. One concern: the helper bails out
  early when `schema.ast.indexSignatures.length > 0`, meaning any
  schema with a record/catchall at the top level loses unknown-key
  rejection. `Config.Info` itself doesn't have a top-level record so
  the current call sites are fine, but a comment documenting that
  invariant — and an assertion in tests — would help future schemas
  that get routed through here.

- `packages/opencode/src/config/permission.ts` removes the dual `InfoZod`
  union (lines ~55–82 in the old file) and the `[ZodOverride]:
  InfoZod` annotation. With `propertyOrder: "original"` now applied at
  decode time, this dedup is correct: previously the Zod override
  existed *only* to preserve key order, which Effect Schema didn't.
  Worth eyeballing one more time that no caller still imports
  `InfoZod` directly (it was an internal symbol so this should be
  safe).

- `packages/opencode/src/config/agent.ts` switches from
  `zod(AgentSchema).transform(normalize)` to a true Effect pipe with
  `Schema.decodeTo(AgentSchema, { decode: SchemaGetter.transform(...),
  encode: SchemaGetter.passthrough({ strict: false }) })`. The encode
  side using `strict: false` is the right call for forward-compat with
  unknown options, but it does mean encoding never validates
  `agent.options` shape — confirm that's intentional and not a
  regression vs. the old Zod path which did re-validate on encode.

- `config.ts` swaps `AgentRef = Schema.Any.annotate({ [ZodOverride]:
  ConfigAgent.Info })` for direct `ConfigAgent.Info` references in the
  `mode`/`agent` struct definitions. The MCP `enabled-only` legacy
  branch also flips from `Schema.Any.annotate({ [ZodOverride]: z.object(
  ... ) })` to `Schema.Struct({ enabled: Schema.Boolean })`. That's
  cleaner and removes a real code-path duplication, but check the
  generated `config.schema.json` diff: `Schema.Struct` without an
  explicit "no additional properties" flag might emit a slightly
  different JSON Schema than the old strict zod object did, which can
  break editor autocomplete on `mcp.<name> = { enabled: false }`.

- The two `ConfigParse.schema(Info.zod, ...)` → `ConfigParse.effectSchema(
  Info, ...)` swaps in `config.ts:362` and `config.ts:754` (write-back
  path) are the actual "preserve permission order" surface. Worth a
  test that round-trips a config with `permission: { bash: "ask",
  edit: "deny", "*": "allow" }` and asserts the decoded object keeps
  that exact ordering.

## Rationale

Real bug fix with a clean schema-layer dedup. The `propertyOrder:
"original"` flip is the right primitive to use; previously the Zod
override was a workaround for exactly this gap. Nits are non-blocking:
add (a) a regression test on permission key order, (b) a comment in
`topLevelExtraKeys` documenting the index-signature carve-out, and (c)
a quick check of the generated JSON Schema for the MCP enabled-only
branch. Otherwise mergeable.

## What I learned

When a config layer has first-match-wins semantics, key insertion order
is part of the public contract. If your schema library reorders keys
during decode (most do, since they iterate the schema's known fields
first), you must either pin order at decode time (Effect's
`propertyOrder: "original"`) or carry your own raw-keys side-channel.
The bug fixed here is exactly that mismatch.
