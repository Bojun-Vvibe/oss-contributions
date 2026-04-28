# sst/opencode #24693 — fix(httpapi): match legacy session query parsing

- **PR**: [sst/opencode#24693](https://github.com/sst/opencode/pull/24693)
- **Head SHA**: `8f8a976c`

## Summary

Closes the parity gap between the experimental HttpApi session routes and the
legacy `/session` route around query-string boolean parsing. Prior `roots` /
`archived` query fields were typed as `Schema.Literals(["true", "false"])`,
which Effect rejected with a 400 for any other value (`?roots=1`,
`?roots=false`, omitted-but-present-as-empty), while the legacy route accepted
"any present non-empty value as truthy". This widens both schemas to
`Schema.optional(Schema.String)` and routes through a shared
`legacyQueryBoolean(value) => value ? true : undefined` helper so the wire
contract matches verbatim.

## Specific findings

- `packages/opencode/src/server/routes/instance/httpapi/experimental.ts:54-66`
  — schema widened (`roots`, `archived` → `Schema.optional(Schema.String)`),
  with the new `legacyQueryBoolean` helper at `:64-66` and call sites at
  `:311,317`. Helper is intentionally trivial (`value ? true : undefined`)
  which matches the legacy `Session.list` arg shape — passing `false` instead
  of `undefined` would change downstream filter semantics, so the
  `undefined`-on-falsy branch is load-bearing, not a stylistic choice.
- `packages/opencode/src/server/routes/instance/httpapi/session.ts:39-49,88-91,440`
  — same helper is duplicated locally rather than imported. Two reasonable
  positions: (a) keep duplicated since the routes are independently versioned
  and shouldn't cross-import internals, (b) hoist to a shared `httpapi/util.ts`.
  The PR picks (a) which is fine, but the duplication should at least carry an
  inline `// keep in sync with experimental.ts` comment so a future divergence
  doesn't slip through.
- `session.ts:476-479` — `before` guard tightened from
  `ctx.query.before !== undefined` to `ctx.query.before` (truthy). Subtle
  semantic shift: `?before=` (empty string) used to be a 400, now it's
  treated as "no cursor" and routes through the no-cursor list path. Test
  coverage at `httpapi-json-parity.test.ts:135-139` (`session.messages empty
  before`) pins this new behavior against the legacy route — good, the parity
  harness is the right place for this assertion.
- Test additions at `test/server/httpapi-json-parity.test.ts:107-138` — exercises
  `?roots=false`, `?roots=1`, and `?before=` for both `session.list` and
  `experimental.session`. Each entry rides through the existing
  `expectJsonParity` envelope which compares the two route stacks
  endpoint-by-endpoint, so a future regression where one path diverges produces
  an `expect({label,body}).toEqual({label,body:legacy})` diff that names the
  exact endpoint.

## Nits / what's missing

- Helper duplication (`legacyQueryBoolean`) has no cross-reference comment
  between the two files. One-line `// keep in sync with experimental.ts` would
  prevent silent divergence.
- The truthy semantic for `?roots=` (empty value → falsy → `undefined`) isn't
  pinned by a test — only `?roots=false` and `?roots=1` are exercised. Empty
  string is the most likely accidental shape from a JS client building a
  URLSearchParams with a `null`-coerced value, so worth one extra row.
- No PR-body callout that this changes the `?before=` guard semantics. Behavior
  shift from 400 → 200-with-no-cursor is small but observable to clients that
  were relying on the validation error.

## Verdict

**merge-as-is** — surgical schema-widening that closes a real parity gap with
a load-bearing helper rebuilt at the right shape for the underlying
`Session.list` API, three new parity-harness rows, and the parity envelope's
endpoint-labeled diff catches any future divergence by construction. The nits
(comment cross-reference, empty-string-roots row, before-guard PR-body callout)
are real but small enough to land as a follow-up rather than block.
