# sst/opencode #24691 — feat(agent): add order field for configurable agent cycling order

- URL: https://github.com/sst/opencode/pull/24691
- Head SHA: `cd9645d5906e2b5fad3fc84e89dae41f95a84a61`
- Size: +83 / -1 across 3 files
- Verdict: **merge-as-is**

## What the change does

Closes a real ordering complaint (issue #7372): plugins like oh-my-opencode
ship multiple agents in a deliberate sequence (Sisyphus → Hephaestus →
Prometheus → Atlas) and the existing alphabetical sort at `Agent.list` mangled
that into Atlas → Hephaestus → Prometheus → Sisyphus.

Three edits:

1. `packages/opencode/src/config/agent.ts:38-41` adds an optional `order`
   field on `AgentSchema` typed as `PositiveInt` with a self-documenting
   description ("Lower values appear first. Agents without order are sorted
   alphabetically after ordered agents.").
2. `packages/opencode/src/config/agent.ts:67` adds `"order"` to
   `KNOWN_KEYS` so config-validation does not flag it as unknown.
3. `packages/opencode/src/agent/agent.ts:36` adds `order:
   Schema.optional(Schema.Number)` to `Info`, `:260` adds the merge rule
   `item.order = value.order ?? item.order` (last-non-undefined-wins, matching
   the surrounding fields' pattern), and `:293` adds the new sort key
   `[(x) => x.order ?? Infinity, "asc"]` between the default-agent-pin and the
   alphabetical fallback.

## What is load-bearing

- `?? Infinity` is the right sentinel: agents without `order` collate **after**
  every ordered agent, in stable alphabetical order, which is exactly the
  documented contract and matches the test at `agent.test.ts:454-491` (alpha
  order=1 → beta order=2 → gamma → zebra).
- The sort key ordering is correct: default agent first (preserved from prior
  behaviour), then `order` ascending, then name ascending as the tiebreaker.
  Crucially the default-agent pin runs *before* the order check, so the test
  at `:419-453` asserting "default agent first regardless of order" passes
  even when the default has `order: 2` and `alpha` has `order: 1`.
- `PositiveInt` (not generic `Number`) at the config schema layer at `:38`
  rejects `0` and negative values — sane because `0` would be ambiguous with
  "unset" and negatives would silently sort before the default-agent pin if
  the pin logic ever changed. Picking `PositiveInt` is the conservative call.

## Test coverage

Two new tests at `test/agent/agent.test.ts`:
- `:419-453` "Agent.list respects order field for sorting" — three agents
  ordered 1/2/3, asserts default-agent-first invariant holds.
- `:454-491` "Agent.list sorts agents without order after ordered agents" —
  mixed ordered/unordered, asserts ordered come first, unordered fall back to
  alphabetical.

The original alphabetical-only test was renamed at `:394` ("...when no order
is set") rather than deleted — good, that's the regression anchor for the
no-order-set case.

## Nits

None worth blocking on. Two minor things if a follow-up touches this code:

- The merge-rule `item.order = value.order ?? item.order` at `:260` means an
  upstream config that sets `order: 5` and a downstream override that omits
  `order` will keep `5`, which is the right shape, but a downstream that
  wants to **clear** `order` cannot — that's consistent with how every other
  field in the merge block behaves so no action needed.
- Doc gap: the new field could be mentioned in the agent-config docs page
  with a one-line example. Not blocking.

## Recommendation

Ship. Backward-compatible additive field with PositiveInt validation,
correct sort-key placement, two purpose-built tests, original test preserved
as the no-order regression anchor.
