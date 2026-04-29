# sst/opencode#24979 — refactor(question): rename bus binding

- **PR**: https://github.com/sst/opencode/pull/24979
- **Head SHA**: `563d8202fbea2fc2f37f0a89b9f60f71b0309b30`
- **Size**: +4 / −4, 1 file
- **Files**: `packages/opencode/src/question/index.ts`

## Context

Inside the Question service `Layer.effect` factory at `packages/opencode/src/question/index.ts:135`, the local binding `const bus = yield* Bus.Service` is renamed to `vehicle`, and the three call sites at `:172`, `:194`, `:211` (`bus.publish(Event.Asked, ...)`, `bus.publish(Event.Replied, ...)`, `bus.publish(Event.Rejected, ...)`) update accordingly. Pure local rename — no imports, types, exports, or behavior change.

## Why this exists / problem solved

The PR body says "Rename the local Question service Bus binding from `bus` to `vehicle`." There is no linked issue, no claim of a real problem, and no description of why `vehicle` is a better name than `bus` for "an event bus instance". The only narrative the diff offers is "I renamed it." The author also notes verification is missing: `bun typecheck` failed in their env because `tsgo` was unavailable, so we don't even have a green typecheck on the renamed binding (though it's a 4-occurrence local rename so the risk is essentially zero).

## Design analysis

The rename is mechanically correct — every reference inside the `Layer.effect` callback is updated, no dangling `bus.publish` remains, and no other variable is shadowed. The new identifier `vehicle` is, however, *semantically worse* than `bus`:

1. The imported module is literally `Bus` (the class is `Bus.Service` from `import { Bus } from "../bus"`). The local binding `bus` was the canonical lowercased form of the type — `vehicle` introduces a synonym that the reader has to mentally translate back to "this is the Bus service" every time.
2. "Vehicle" is the parent metaphor of "bus", so the rename moves *up the abstraction ladder* (less specific) rather than down (more specific). For a local binding inside a factory whose job is to publish to that exact bus, the more specific name carries more information.
3. `bus` is not shadowed anywhere in this file (verified by reading the diff context — the outer scope has no `bus` binding), so there's no naming-collision motivation.

Looking at the call shape, the publishes are also straight-line (`yield* vehicle.publish(Event.Asked, info)` at `:172`) — there's no chance of the rename interacting with effect composition or with any `yield*` semantics. So the diff is safe; it's just *unmotivated*.

## Risks

- Effectively zero runtime risk. Local binding rename in a single file with all call sites updated.
- Minor reviewer/maintainer cost: future contributors reading `vehicle.publish(...)` will need to grep upward to discover it's the event bus, vs. `bus.publish(...)` which is self-documenting.
- The PR author admits typecheck didn't run in their environment, so the only real verification is "I read the diff" — for a 4-line local rename this is acceptable but worth flagging.

## Suggestions

1. Either drop this PR entirely (the original `bus` name was clearer) or expand the PR body with the actual motivation. If the rename is part of a larger upcoming refactor where `bus` becomes a parameter name in an outer scope and would shadow this local, say so — that's a legitimate reason. If it's stylistic preference, that's not a strong-enough reason to merge a name change in a public OSS repo.
2. If kept, also rename the comparable bindings in sibling Effect-style services — e.g., any other `Layer.effect` that does `const bus = yield* Bus.Service` — so the codebase doesn't end up with a half-renamed convention. A quick grep for `yield\* Bus\.Service` will surface the set.
3. Add the typecheck output (or a CI-link) before merge so the "tsgo wasn't available locally" gap is closed; for a 4-line rename this is a 30-second sanity check, not a real ask.

## Verdict

**needs-discussion** — diff is mechanically safe but the rename is unmotivated and arguably worsens readability. Maintainer should either close with a "we prefer `bus` here" or accept after the author justifies why `vehicle` is clearer than `bus` for a binding to `Bus.Service`. Not `request-changes` because there's no defect; not `merge-as-is` because the change is a regression in name quality.

## What I learned

A 4-line rename PR is a useful test of "does the project care about naming taste?" In Effect-style code where services are imported by their PascalCase identity (`Bus.Service`, `Storage.Service`, etc.), the convention of binding them locally as the lowercased identifier (`const bus = yield* Bus.Service`) carries information for free — it tells the reader exactly which service they're calling without re-deriving it from the type. Synonym-renames break that convention for no behavioral gain. The right answer for a maintainer is usually "thanks, but we'd like to keep the type-name correspondence" — and that's an example of the kind of informal style guideline that's worth writing down in CONTRIBUTING.md if it isn't already, because PRs like this one will keep arriving otherwise.
