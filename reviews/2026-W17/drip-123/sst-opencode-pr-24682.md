# sst/opencode#24682 — test(httpapi): cover session json parity

- **PR**: https://github.com/sst/opencode/pull/24682
- **Author**: kitlangton (Kit Langton)
- **Head SHA**: `cd47262081e8ced842ddd210f44ec70ea1f0edd0`
- **Verdict**: `merge-as-is`

## Context

This PR is the test-side complement to the `optionalOmitUndefined` fix that landed via #24671 last drip. The session-info schema migration moved several historically-`undefined`-bearing fields (`summary.diffs`, `revert.partID`, `revert.snapshot`, `revert.diff`) onto `optionalOmitUndefined(...)`, but the only existing regression coverage at `packages/opencode/test/session/session-schema.test.ts` was the *top-level* `parentID` / `project.name` omit-on-undefined check from the original migration. Nested-object omit semantics were untested, and the entire HTTP-API surface (the experimental route stack that consumes the same encoded JSON) had no parity contract against the legacy route stack at all. This PR closes both gaps.

## What it changes

Two test additions plus a comment-only docstring touch-up at `packages/opencode/src/util/schema.ts:8-13` (clarifies that `optionalOmitUndefined` accepts `undefined` "on the type side" but encodes it as an omitted key — a wording win, no behavior change). The new nested-omit test at `test/session/session-schema.test.ts:167-188` constructs a `Session.Info` with `summary.diffs = undefined` plus a `revert: { messageID, partID: undefined, snapshot: undefined, diff: undefined }` payload, runs it through `Schema.encodeUnknownSync(Session.Info)`, and asserts `Object.hasOwn(encoded.summary, "diffs") === false` plus a loop over `["partID","snapshot","diff"]` asserting the same on `encoded.revert`. The much larger addition is `test/server/httpapi-json-parity.test.ts:1-148`: a single-test-case parity harness that boots the route stack twice (with `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI` toggled), seeds a parent + child session + one user message + one text part via the `Session.Service` Effect layer, and walks seven endpoints — `session.list?roots=true`, `session.list`, `session.get`, `session.children`, `session.messages`, `session.message`, and the experimental `session` listing — asserting `expect({label, body}).toEqual({label, body: legacy})` on each. The `{label, body}` envelope is a quietly-clever bun:test ergonomic — it makes diff output point straight at the failing endpoint instead of dumping a 200-line JSON shape with no header.

## Design analysis

The harness avoids three foot-guns I'd expect a shallower attempt to hit. First, it `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI = original` in `afterEach` (`:113-117`) instead of leaving the flag mutated for whatever runs next in the same bun-test worker — flag-scoped state leaking across files is a perennial source of flakes in the opencode test suite. Second, the sequential `.reduce(promise, input => promise.then(...))` chain at `:143-146` is the right primitive for a parity test against shared `Instance` state — `Promise.all` would have raced seven concurrent `Instance.provide` calls against the same `tmp.path` with `resetDatabase()` in `afterEach` and produced nondeterministic 200-vs-500 splits. Third, `pathFor` at `:51-53` reduces the path-template `:sessionID` / `:messageID` placeholders before request-time so the assertion message matches the actual URL hit, not the template. The `seedSessions` helper at `:55-83` correctly threads `MessageID.ascending()` / `PartID.ascending()` through the `updateMessage` / `updatePart` calls — using `.ascending()` keeps the seeded IDs lex-sortable so any pagination shape difference between the two route stacks would surface deterministically.

## Risks

Low. The parity test is read-only against seeded state and runs entirely in `tmpdir({git: true})` so there's no host-state leak. One latent concern: the test asserts byte-for-byte JSON parity via `toEqual`, which means any future intentional shape divergence between the experimental and legacy stacks (e.g. adding a field on the experimental side first behind the flag) will require updating this test in the same PR. That's a feature, not a bug — the whole point is to make divergence impossible to land silently — but reviewers of those future PRs should expect to touch `httpapi-json-parity.test.ts` and not treat the diff as a flake. The seven-endpoint enumeration is also a surface contract: if a new session-read endpoint is added without being added here, parity drift can accumulate undetected. A `// keep in sync with SessionPaths` comment at `:127` would make the contract explicit but isn't blocking.

## Suggestions

None blocking. If a follow-up wants to harden further: (1) add a `session.list?roots=true&limit=1` case to exercise the pagination shape, (2) fold the `["partID","snapshot","diff"]` loop into a parameterized `test.each` so a future field addition shows up as a discrete test name in the report, and (3) consider asserting that the `encoded.revert.messageID` survives the encode (the current test only proves the omit-when-undefined contract, not that the non-undefined sibling field still round-trips — though that's covered transitively by the parity test). All three are nice-to-have, not blockers.

## What I learned

The `{label, body}` envelope-on-`toEqual` pattern is a small but durable improvement over the more common "loop and assert each endpoint with a manual `expect().withContext()`" — it gives you the right diff output for free without a custom matcher, and it makes the test reproducible (one failure shows the failing label inline). Worth stealing for any "N variants of the same shape contract" test in any project.
