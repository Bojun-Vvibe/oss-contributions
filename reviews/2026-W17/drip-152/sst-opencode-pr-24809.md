# sst/opencode#24809 — fix(httpapi): document instance query parameters

- **Repo:** [sst/opencode](https://github.com/sst/opencode)
- **PR:** [#24809](https://github.com/sst/opencode/pull/24809)
- **Head SHA:** `9a70c691a6c11b62bfbdd905bbb51951449d1d0c`
- **Size:** +88 / -1 across 3 files (`pty.ts`, `public.ts`, `httpapi-bridge.test.ts`)
- **State:** MERGED

## Context

Two parity gaps between the legacy Hono OpenAPI generator and the new
Effect HttpApi generator: (1) every instance route accepts the shared
`directory` and `workspace` query params on the legacy side but the
Effect side never declared them — so legacy SDK consumers paginating
multi-workspace instances saw the params, Effect SDK consumers didn't,
and code-generated clients diverged in shape. (2) Symmetrically, the PTY
websocket `connect` route declared a `cursor` query on the Effect side
that legacy never exposed in the contract surface — so the Effect spec
was strictly *broader* in one direction and strictly *narrower* in the
other. PR body claims OpenAPI parameter mismatches drop from 107 → 0
and SDK signature mismatches drop from 111 → 43.

## Design analysis

Three distinct surfaces:

1. **PTY narrowing** at `pty.ts:118-121`: drops the `query: CursorQuery`
   declaration from the `connect` endpoint. The cursor param is still
   accepted at runtime by the websocket handler — this is purely a
   contract-surface delta to bring the Effect spec in line with what
   legacy advertises. Right shape: the websocket implementation owns the
   cursor semantics, the contract just shouldn't lie about it.

2. **Instance broadening** via post-processor at `public.ts:34-66`. The
   `documentInstanceQueryParameters` transform runs at OpenAPI emit
   time, walks every path, and (for non-`/global/`/non-`/auth/` paths)
   prepends two synthetic `directory` and `workspace` parameter
   declarations, then concatenates whatever existing query params the
   route already declared *after filtering out any that collide by name*
   (`:60-62`). The collision filter is load-bearing — without it any
   route that re-declares `directory` natively would emit a duplicate
   parameter entry which downstream OpenAPI validators reject as
   spec-invalid.

   The path-prefix exclusion `/global/` and `/auth/` is the right
   discriminator (those are the two surfaces that don't accept the
   instance-scoping params) but it's a string-match policy living in
   the OpenAPI emitter rather than declared on the API definitions
   themselves — so a future `/system/` or `/health/` prefix that's
   also instance-agnostic will silently get the synthetic params
   declared on its routes until someone notices.

3. **Parity test** at `httpapi-bridge.test.ts:37-115`. New
   `openApiParameters` extractor walks both Hono and Effect OpenAPI
   docs, builds a `Map<"METHOD /path", string[]>` keyed by sorted
   `${in}:${name}:${required}` triples, and asserts the set-difference
   is empty. The triple-keying on `(in, name, required)` is the
   correct equivalence relation — two specs that agree on parameter
   *names* but disagree on whether one is required would falsely pass
   a name-only check. The `parameterKey` helper at `:55-59` defends
   against malformed params (missing `in` or `name`) by returning
   `undefined` and filtering them out at `:48` rather than crashing —
   so a future spec change that adds a non-standard parameter shape
   still gets the test failing on the *real* mismatches without being
   masked by a parser exception.

## Risks / nits

1. **Silent over-application of the synthetic params.** The
   `path.startsWith("/global/") || path.startsWith("/auth/")` exclusion
   at `public.ts:55` is a denylist not an allowlist. Any new
   instance-agnostic path prefix added later will inherit the synthetic
   `directory`/`workspace` declarations until a maintainer remembers to
   extend the denylist. Better shape: either declare an `instanceScoped`
   bit on each `HttpApi.make("...")` call and have the transform read
   it, or invert to an allowlist of known-instance prefixes. Same
   bug-shape as classic CORS denylist drift.

2. **Loss of generic typing in the transform.** `input as OpenApiSpec`
   at `:51` and the locally-defined `OpenApiParameter`/`OpenApiOperation`
   at `:20-32` duplicate types that the `effect/unstable/httpapi` package
   likely already exports (`OpenApi.Spec`/`OpenApi.Operation`). If the
   upstream Effect API ever evolves the parameter shape (adds `style`,
   `explode`, `allowEmptyValue`, etc.), the locally-defined types will
   silently drop those fields when `documentInstanceQueryParameters`
   round-trips through them. Worth importing the upstream types.

3. **Parity test is ordering-sensitive but not declared as such.** The
   test sorts parameter keys at `:46` so order-only diffs don't trip
   it, but the production transform at `public.ts:58-63` puts the
   synthetic params *first* via spread order. If a downstream consumer
   relies on parameter order (some OpenAPI codegens do, particularly
   for path/query/header section grouping), the test wouldn't catch
   a regression where Hono and Effect agreed on the *set* but
   disagreed on the *order*. Document the policy in a comment.

4. **PTY narrowing has no parity test.** The `cursor`-removal at
   `pty.ts` is asserted only via the existing parity test at `:64-77`
   (the route-key equality check), not via a positive assertion that
   the `connect` route declares zero query params. A future re-add of
   `query: CursorQuery` would pass the parameter-parity test (since
   Hono also doesn't declare it, both surfaces would drop it from the
   diff bucket), but should still be guarded against accidental
   re-introduction.

## Verdict

**merge-after-nits.** The fix is correctly minimal — declarative
post-processor for the broadening case, simple removal for the
narrowing case, and a parity test that asserts the strengthened
contract. Pre-merge ask: convert nit 1 (denylist → allowlist or
per-API `instanceScoped` flag) and nit 2 (use upstream Effect types).
Nits 3 and 4 are follow-ups.

## What I learned

- OpenAPI-spec parity between two parallel server implementations is
  best asserted as a set-difference on `(method, path, in, name,
  required)` tuples — the four-tuple is the natural key for "is this
  parameter declared the same way on both sides," and sorting per-path
  before diffing avoids false-positive ordering noise.
- Post-processor transforms that mutate generated OpenAPI specs to
  add/remove parameters are a powerful escape hatch for cases where
  the source-of-truth schema language can't express a cross-cutting
  concern (here: "every instance route accepts these two query
  params"), but they trade declarative-clarity for centralization —
  a denylist of "paths that don't apply" sitting in the emitter is
  easier to forget about than the same metadata living on each API
  definition.
- The `(in, name, required)` triple is the equivalence relation that
  matters for parameter-parity. Name-only equivalence misses
  required/optional drift, and full-schema equivalence is too strict
  for the parity-as-contract use case.
