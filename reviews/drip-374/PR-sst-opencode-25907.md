# sst/opencode #25907 — refactor: remove unused DynamicDescription hack in tool.ts

- URL: https://github.com/sst/opencode/pull/25907
- Head SHA: `70712fd7cf119885f26b27e4bcbe473a4f1813e3` (`70712fd`)
- Author: @srivenkateswaran6002
- Diff: +0 / −3 (1 file)
- Verdict: **merge-as-is**

## What this changes

Pure deletion. `packages/opencode/src/tool/tool.ts` lines 12-13 (in the
pre-patch file) are removed:

```ts
// TODO: remove this hack
export type DynamicDescription = (agent: Agent.Info) => Effect.Effect<string>
```

That's it — no other files touched, no test changes, no behavior
delta. The author confirms (a) global codebase grep for
`DynamicDescription` returns zero usages outside this declaration, and
(b) `bun run typecheck --filter=opencode` is clean.

## Why this is the right shape

The exported alias was effectively a typed promise to the rest of the
codebase that "we are still planning to plumb dynamic per-agent
description strings through tool definitions" — and it's been sitting
unused for long enough that the author who wrote it left a literal
`TODO: remove this hack` comment on the line above. Removing dead
exports of this shape has two real wins beyond LOC: (1) it stops
tooling like `tsc --build` from generating the type into emitted
`.d.ts` files where downstream consumers would otherwise discover and
use it before the maintainers had decided whether the contract is
real, and (2) it deletes a half-baked `Agent.Info → Effect<string>`
shape that future contributors would inevitably look at and try to
"finish" by wiring it into `Tool.Definition`, propagating a partially-
designed concept.

## Concerns

None blocking. The PR title and body are accurate; the change is
correctly scoped (single file, single export, no orphaned imports).
The author notes they "accidentally closed the linked issue (#25906)
and don't have permissions to reopen it" — reviewer should glance at
#25906 before merge to confirm the issue text doesn't ask for
something larger than this 3-line deletion (e.g. an actual
implementation), but going by the title and the author's analysis the
intent is exactly "remove the dead type alias."

Canonical small-deletion `merge-as-is`.
