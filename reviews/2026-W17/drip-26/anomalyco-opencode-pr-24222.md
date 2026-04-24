# anomalyco/opencode PR #24222 — fix permission config order

- **Repo:** anomalyco/opencode
- **PR:** [#24222](https://github.com/anomalyco/opencode/pull/24222)
- **Head SHA:** `6e4f4544014e05823b8475c2a862eadbdf7c2cc8`
- **Author:** thdxr (Dax)
- **Size:** +40 / −70 across 4 files
- **Reviewer:** Bojun (drip-26)

## Summary

Reverses a recently-added "wildcard rules sort first" canonicalisation
in `Permission.fromConfig` and replaces the
schema-driven `StructWithRest` decoder for permission config with a
custom Zod schema (`InfoZod`) that preserves the user's literal key
order through parse → fromConfig → evaluate. The end-state semantics
become: **permission precedence follows the order the user wrote in
their config**, full stop. Combined with `evaluate()`'s `findLast`,
this means a later wildcard *can* override an earlier specific rule
— which is the opposite of what the prior canonicalisation enforced.

Net diff is `-30` lines across the package (`-67` in source, `+15`
in tests, plus a new `+22` Zod definition). Most of the deletion is
the now-removed "sort wildcards first" guard plus the docstrings
that justified it.

## Key changes

### `packages/opencode/src/config/permission.ts:21–67`

The `// Known permission keys get explicit types …` block is rewritten
end-to-end. Old behaviour:

> StructWithRest canonicalises key order on decode (known first, then
> rest), which used to require the `__originalKeys` preprocess hack
> because `Permission.fromConfig` depended on the user's insertion
> order. That dependency is gone — `fromConfig` now sorts top-level
> keys so wildcard permissions come before specifics, making the
> final precedence order-independent.

New behaviour:

> Known permission keys get explicit types in the Effect schema for
> generated docs/types. Runtime config parsing uses `InfoZod` below
> so user key order is preserved for permission precedence.

The new `InfoZod` (`permission.ts:58–67`):

```ts
const ACTION_ONLY = new Set(["todowrite", "question", "webfetch", "websearch", "codesearch", "doom_loop"])

const InfoZod = z
  .union([zod(Action), z.record(z.string(), z.union([zod(Action), z.record(z.string(), zod(Action))]))])
  .transform(normalizeInput)
  .superRefine((input, ctx) => {
    for (const [key, value] of globalThis.Object.entries(input)) {
      if (!ACTION_ONLY.has(key) || typeof value === "string") continue
      ctx.addIssue({ code: "custom", message: `${key} must be a permission action`, path: [key] })
    }
  })
```

…and it's wired in via `.annotate({ [ZodOverride]: InfoZod })` on
the existing Effect schema, so the Effect-derived JSON schema /
types stay intact for docs and TypeScript consumers, but the
runtime parser used by config loading honours user key order.

### `packages/opencode/src/permission/index.ts:288–300`

`fromConfig` deletes the wildcard-first sort:

```ts
// before:
const entries = Object.entries(permission).sort(([a], [b]) => {
  const aWild = a.includes("*")
  const bWild = b.includes("*")
  return aWild === bWild ? 0 : aWild ? -1 : 1
})
const ruleset: Ruleset = []
for (const [key, value] of entries) { ... }

// after:
const ruleset: Ruleset = []
for (const [key, value] of Object.entries(permission)) { ... }
```

Pure deletion; iteration is now over the parsed object's natural
key order, which `InfoZod` no longer canonicalises.

### `packages/opencode/test/config/config.test.ts:1495–1535`

The existing test was asserting the *old* canonicalisation
(known-first, rest-in-insertion-order). It's replaced with a test
that asserts user insertion order is preserved literally — for
the same input fixture, the expected `Object.keys(...)` array
flips from `["read", "edit", "external_directory", "todowrite",
"*", "write", "thoughts_*", ...]` to `["*", "edit", "write",
"external_directory", "read", "todowrite", "thoughts_*", ...]`.

### `packages/opencode/test/permission/next.test.ts`

Five tests rewritten to match the new semantics:

- `fromConfig - specific key beats wildcard regardless of JSON key order`
  → `fromConfig - preserves top-level config key order` (now asserts
  the *opposite* — `{ "*": "deny", bash: "allow" }` evaluates `bash`
  to `allow`, but `{ bash: "allow", "*": "deny" }` evaluates `bash`
  to `deny`).
- The `top-level ordering: wildcards first, specifics after` test is
  inverted to `top-level ordering is not sorted by wildcard
  specificity` and asserts `["bash", "*", "edit", "mcp_*"]` is the
  literal observed order.
- `evaluate - permission patterns sorted by length regardless of
  object order` is renamed to `later wildcard permission can
  override earlier specific permission` to reflect the new contract.

## What's good

- The **annotate-with-`ZodOverride`** pattern is the right move for
  this codebase: keep the Effect schema as the source of truth for
  generated types and docs, but allow a hand-rolled Zod parser when
  Effect's `StructWithRest` inherent canonicalisation is the wrong
  semantic for the runtime use. That's a clean separation of
  "schema for tooling" vs "schema for runtime".
- The `superRefine` over `ACTION_ONLY` keeps the typed-key
  validation that `StructWithRest` previously gave for free
  (`todowrite`/`question`/`webfetch`/`websearch`/`codesearch`/`doom_loop`
  must be string actions, not nested objects). Without this the
  switch to `z.record(z.string(), ...)` would have silently
  accepted nonsense like `todowrite: { "foo": "allow" }`.
- The test rewrites in `test/permission/next.test.ts` are honest
  — they don't try to preserve the old assertions while changing
  the implementation; they actually invert the expectations. That
  makes the semantic flip auditable in the diff alone.

## Concerns

1. **This is a breaking change in user-facing semantics.** Anyone
   who has a config like

   ```json
   { "*": "deny", "bash": "allow" }
   ```

   relying on the previous "wildcards-first canonicalisation, so
   `bash` overrides `*`" behaviour will get the *opposite* result
   after this PR (the `*: deny` would have come first then `bash:
   allow` would override; that still works). But:

   ```json
   { "bash": "allow", "*": "deny" }
   ```

   used to evaluate `bash` to `allow` (sort moved `*` to the front)
   and now evaluates `bash` to `deny` (literal order, `findLast`
   picks `*`). Existing user configs that happen to be in the second
   shape will silently change behaviour. **There is no migration
   note or warning in the diff.** This needs to be called out
   prominently in the PR description and ideally in release notes.

2. **The doc string change is internal.** The new comment
   ("Permission precedence follows the order users write in
   config, so parsing must not canonicalise known keys ahead of
   wildcard or custom keys") explains the *implementation*, not the
   *user contract*. The user-facing docs in
   `docs/permissions.mdx` (referenced by the deleted "canonical
   documented example unchanged" test name) almost certainly still
   document the old "wildcards-as-fallback" mental model. **Block
   on confirming the user docs match the new semantics** — the
   deleted test was named `documented example unchanged` and the
   replacement is named `documented fallback-first example`,
   which strongly implies the docs assume the user puts wildcards
   *first*. That's a real shift in advised config style.

3. **`globalThis.Object.entries` at `permission.ts:63`** — the
   `globalThis.` prefix is unusual and presumably fights a
   schema-internal `Object` shadow. Worth a one-line comment noting
   *why* (`// Effect re-exports `Object`; reach for the platform
   Object explicitly`) so the next reader doesn't "simplify" it
   back and break the parse.

4. **The `ZodOverride` annotation is a runtime-only escape hatch.**
   The Effect schema still claims `StructWithRest` semantics, so
   anything in the codebase that introspects the schema (codegen,
   docs, tooling) will produce types/docs that contradict runtime
   behaviour for permission config specifically. If that mismatch
   matters elsewhere (e.g. JSON-schema generation for editor
   completion), the fix is incomplete. Not visible in this diff;
   needs eyes on the schema-consuming sites.

5. **Nit:** `ACTION_ONLY` is a hard-coded set repeated outside the
   schema. The known-key list in `InputObject` (`read`, `edit`,
   `bash`, etc.) and the `ACTION_ONLY` set are now two parallel
   key catalogues that have to stay in sync. Worth deriving
   `ACTION_ONLY` from a tagged Effect schema field, or at minimum
   colocating the two with a `// keep in sync with InputObject
   declaration above` comment.

## Risk

Medium. The implementation is small and the tests are honest, but
this changes documented user-facing semantics for a config surface
that existing users have already written against. The migration
story is the unaddressed risk.

## Verdict

**needs-discussion**

The implementation is sound, but the *direction* of the semantic
change deserves an explicit ADR-style call in the PR description:

- Why are we reverting the "sort wildcards first" guard added in
  the not-too-distant past?
- What's the migration path for existing user configs that depended
  on it?
- Does `docs/permissions.mdx` still recommend the old key ordering?
- Should there be a one-release deprecation warning ("your config
  has a wildcard permission listed after a specific permission;
  in the next release the specific permission will no longer
  override the wildcard") before the silent flip?

If the answer to #2 is "we'll update docs and ship a release note,
existing configs that read top-to-bottom-style are unchanged",
then this becomes `merge-after-nits` (block on docs update plus
nits 3 and 5). If the answer is "we'll add a deprecation pass
first," that's another PR.

## What I learned

`StructWithRest` and "user-key-order-significant runtime semantics"
fundamentally don't compose. Once your config grammar makes
*ordering* part of its observable behaviour, you cannot use a
schema decoder that canonicalises keys, even if it canonicalises
them "compatibly". The right fix is the one this PR makes — keep
the canonicalising schema for typing/docs, hand-roll a
order-preserving parser for the runtime path, and connect the two
through an explicit annotation. The wrong fix would have been
"sort the keys in `fromConfig` to compensate" — which is exactly
what was tried before and is being reverted now.
