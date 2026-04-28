# sst/opencode #24691 — feat(agent): add order field for configurable agent cycling order

- **PR**: [sst/opencode#24691](https://github.com/sst/opencode/pull/24691)
- **Head SHA**: `cd9645d5`

## Summary

Adds an optional `order: PositiveInt` field to the agent config schema so users
can pin Tab-cycling order rather than depending on alphabetical fallback. Default
agent still gets first slot regardless of `order` (preserving prior contract);
agents with `order` set sort by ascending order; agents without `order` sort
alphabetically *after* all ordered agents (`Infinity` sentinel in the comparator).

## Specific findings

- `packages/opencode/src/agent/agent.ts:36` — `order: Schema.optional(Schema.Number)`
  added to the runtime `Info` schema. Note this is `Schema.Number` (any number)
  on the runtime side but `PositiveInt` in the config-side schema at
  `packages/opencode/src/config/agent.ts:38` — asymmetry is intentional (the
  config schema validates user input, the runtime schema is permissive for
  programmatic agents) but worth a one-line comment so a future "make these
  match" cleanup doesn't drop the `PositiveInt` constraint and let `order: -1`
  through the config path.
- `agent.ts:260` — merge path `item.order = value.order ?? item.order` follows
  the same pattern as adjacent fields (`color`, `hidden`, `name`). Correct.
- `agent.ts:288-294` — comparator chain in `sortBy` is the load-bearing piece:
  `[(x) => x.name === default_agent ? "desc" : ..., "desc"]` first slot keeps
  default agent pinned, then `[(x) => x.order ?? Infinity, "asc"]` sorts
  ordered agents by their value, then `[(x) => x.name, "asc"]` is the
  alphabetical tiebreaker. The `?? Infinity` sentinel is the right choice —
  no `order` agents land at the bottom in alphabetical order, exactly what the
  PR description claims and what the second test pins.
- `packages/opencode/src/config/agent.ts:38-41` — `order: Schema.optional(PositiveInt)`
  with a description naming the Tab-cycling semantic. Good — `PositiveInt`
  rejects `0` and negatives at the config layer, so users can't set
  `order: 0` and silently sort before the default agent.
- `config/agent.ts:67` — `"order"` added to `KNOWN_KEYS` so the unknown-key
  warning doesn't fire for it. Easy to miss but correct.
- Test coverage at `test/agent/agent.test.ts:420-466` (order respected with
  default agent first), `:468-491` (mixed ordered+unordered correctly partitions
  ordered-first then alpha-second). Both tests use `Instance.provide` against
  a real tmpdir config, so the path through `loadConfig → AgentSchema validate
  → Agent.list sort` is exercised end-to-end. The default-agent-first invariant
  is pinned by the first test (default `build` has `order: 2` but lands at index
  0 because of the default-agent comparator slot taking precedence over `order`).

## Nits / what's missing

- No regression test for the `order: 0` rejection path — `PositiveInt` should
  reject it but a one-line `expect(loadConfig(...)).toReject()` style assertion
  would pin the contract so a future schema migration that swaps `PositiveInt`
  for `Number` doesn't silently start accepting `0`.
- No coverage for the case where two agents share the same `order` value (e.g.
  both `1`). The comparator falls through to alphabetical, which is the right
  behavior but should be pinned by a test row.
- The runtime `Info.order` is `Schema.Number` (not `PositiveInt`). Since the
  field exists primarily so the comparator can read it, allowing `-1` through
  the runtime API would let programmatic agents sort *before* the default
  agent's natural alphabetical position when the default-agent comparator
  doesn't apply (e.g. when `default_agent` is unset). Edge case but should
  carry an inline comment naming the asymmetry.
- PR description says "agents without order are sorted alphabetically after
  ordered agents" — true for the comparator semantics, but the docstring at
  `config/agent.ts:39-40` reads "appear first" which describes the user-facing
  semantics from the *ordered* agent's perspective. Both correct, just slight
  wording mismatch worth aligning.

## Verdict

**merge-after-nits** — feature design is right (opt-in field, conservative
`PositiveInt` constraint at config boundary, default-agent invariant
preserved by sort-key precedence, two real tests covering both happy paths),
but the runtime-vs-config schema asymmetry and the missing `order: 0`
rejection test should close before merge so the contract stays pinned through
future refactors.
