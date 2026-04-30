---
pr-url: https://github.com/sst/opencode/pull/25032
sha: 9bddf7f3ef53
verdict: merge-as-is
---

# test: cover HttpApi instance context middleware

Adds 167 lines of test coverage at `packages/opencode/test/server/httpapi-instance-context.test.ts:1-167` for the `instanceRouterMiddleware` + `workspaceRouterMiddleware` pair, plus a 15-line `AGENTS.md:1-15` test-style guide for the directory and an 8-line "Testing Patterns" appendix to `.opencode/skills/effect/SKILL.md:31-38`. The test composes middleware in production order (`instanceRouterMiddleware.combine(workspaceRouterMiddleware)`), uses `NodeHttpServer.layerTest` for the in-test server, and probes the `WorkspaceRouteContext`/`InstanceRef`/`WorkspaceRef` contexts via tiny `HttpRouter.add(...)` routes that just echo the resolved IDs — exactly the right "minimum viable surface for routing-layer assertions" shape that lets the test fail loudly when middleware ordering changes without dragging in unrelated route handlers.

The fixture discipline at `:23-37` is particularly clean: the `testStateLayer` flips `Flag.OPENCODE_EXPERIMENTAL_WORKSPACES = true` *and* registers a finalizer that restores the original value, calls `Instance.disposeAll()`, and `resetDatabase()` — three independent global-state hazards each cleaned up explicitly in the same finalizer rather than scattered across teardown hooks. The `Effect.addFinalizer` shape ensures the restore runs even if the test scope errors mid-execution, which is the failure mode that historically left the experimental-workspaces flag in `true` state and broke the next sequential test.

The new `AGENTS.md` deserves its own callout: the rules ("Prefer focused middleware tests with tiny fake routes over full API route trees", "Use `tmpdirScoped({ git: true })` plus `Project.use.fromDirectory(dir)` for project-backed requests", "Add comments for non-obvious test topology") are the correct *prescriptive* form for a test-style guide — they tell the next contributor what to do, not what *not* to do, and they're scoped to this directory so they don't leak global advice. The companion update to `.opencode/skills/effect/SKILL.md:31-38` ("Run tests from package directories such as `packages/opencode`; never run package tests from the repo root") closes the most common newcomer footgun (running `bun test` at root and getting confusing zero-test output).

The only conceivable nit (well below merge-blocking): the test file imports `path` but doesn't appear to use it in the visible portion (line 9) — easy `bun lint` catch but not worth a follow-up commit by itself.

## what I learned
A good test-style `AGENTS.md` per directory is high-leverage — it converts hard-won "this is how testing in this corner of the codebase works" knowledge from oral tradition into a discoverable file that the next contributor (human or LLM) can read in 30 seconds. The two specific moves that make this one work: (1) it's *prescriptive* not *prohibitive* (says what to do, not what to avoid), and (2) it lists the actual helper functions by name (`testEffect`, `it.live`, `tmpdirScoped`, `Project.use.fromDirectory`) so the reader has clickable trailheads instead of having to grep for "what's the right helper here."
