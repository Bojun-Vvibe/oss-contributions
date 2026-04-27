---
pr: 24680
repo: sst/opencode
sha: d6dc71790f1715381c1dd5790d05001710bad7ee
verdict: merge-as-is
date: 2026-04-28
---

# sst/opencode #24680 — fix(cli): resolve --agent names case-insensitively

- **Author**: jeevan6996 (Jeevan Mohan Pawar)
- **Head SHA**: `d6dc71790f1715381c1dd5790d05001710bad7ee`
- **Size**: 39 added / 6 deleted across 3 files (1 new helper, 1 callsite, 1 test).
- **Closes**: #23180

## Scope

`opencode run --agent <name>` required an exact case match on the canonical
agent ID, so `--agent Build` would fail when only `build` was registered.
Fix extracts a tiny `matchAgentName(name, agents) → string | undefined` helper
that prefers an exact match and falls back to a case-insensitive scan,
returning the *canonical* name (not the user input) so downstream code keeps
operating on the registered ID.

## Specific findings

- `packages/opencode/src/cli/cmd/agent-match.ts:5-12` — the helper:
  ```ts
  const exact = agents.find((item) => item.name === name)
  if (exact) return exact.name
  const lowered = name.toLowerCase()
  const match = agents.find((item) => item.name.toLowerCase() === lowered)
  return match?.name
  ```
  Correct shape: exact-first preserves the historical fast path (no surprise
  if two agents differ only in casing — exact wins), and the second pass
  re-scans only on miss. Returning `match?.name` (the canonical) rather than
  the user-supplied `name` is the load-bearing detail — it means downstream
  `Agent.Service.use(...).get(matched)` looks up the right ID even when the
  user typed `BUILD`.
- `packages/opencode/src/cli/cmd/run.ts:586-588` — the `--attach` validation
  branch: `const matched = matchAgentName(name, modes); const agent = matched
  ? modes.find((a) => a.name === matched) : undefined; if (!agent || !matched)
  { ...warn... return undefined }`. The `!agent || !matched` belt-and-braces
  is redundant (if `matched` resolves, the subsequent `find` on the same
  `modes` array with `=== matched` is guaranteed to hit) but it's cheap
  defensive code and reads fine.
- `packages/opencode/src/cli/cmd/run.ts:608-611` — the local resolution
  branch was rewritten from `Agent.Service.use((svc) => svc.get(name))` (an
  exact-match server lookup) to `svc.list()` followed by `matchAgentName(name,
  list)`. This trades one `get` for one `list` round-trip on every `run`
  invocation that doesn't `--attach`. For a small registry that's fine, and
  it's the only way to do case-insensitive matching against user-supplied
  IDs without changing the server API. Worth a small comment naming this
  tradeoff so a future "optimize" PR doesn't revert to `get`.
- `packages/opencode/src/cli/cmd/run.ts:625, 628` — both branches return
  `matched` (the canonical), not `name`. Correct — this is what makes the
  case-insensitive resolution actually flow through to session creation and
  `--attach` enforcement.
- `packages/opencode/test/cli/agent-match.test.ts:5-16` — three tests pin
  exact, case-insensitive (`Build` and `PLAN`), and miss. The case-mixed
  variant `Build` vs `PLAN` covers both "input has caps, registry lowercase"
  and "input all-caps" — the two practical user-error shapes.

## Risk

Low. Pure helper extraction + case-insensitive scan, with both call sites
rewritten symmetrically. The behavior change is strictly more permissive:
inputs that previously errored now succeed; inputs that previously matched
still match exactly first. No public API surface changed (the helper is
internal). Test coverage is appropriate for the surface size.

The one real consideration is the `get(name)` → `list()` swap — for a
deployment with thousands of agents this would matter, but the surface is
inherently bounded by what fits in the agent registry and the call is
already gated behind a CLI invocation so the cost is per-process not
per-request.

## Verdict

**merge-as-is** — small, well-scoped, correctly designed (canonical name
returned, not user input), correctly tested. The `get` → `list` swap could
be commented but isn't blocking.

## What I learned

The "return the canonical, not the input" design choice is the right one
for case-insensitive resolution helpers in general. If `matchAgentName`
returned `name` (the user input) on success, callers would have to remember
to look up by `match.name` themselves, and the next refactor that forgot
this would silently break `--attach` enforcement. Returning the canonical
makes the contract self-enforcing.
