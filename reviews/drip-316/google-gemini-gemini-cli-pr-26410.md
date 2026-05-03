# Review: google-gemini/gemini-cli #26410 — Production readiness review: fix unused imports and security test coverage

- PR: https://github.com/google-gemini/gemini-cli/pull/26410
- Head SHA: `46db34e4296f4a3d4d89d13f284442562abda0e7`
- Author: HaleTom (Tom Hale)
- Size: +0 / -3

## Summary
Removes three lines from `packages/core/src/config/config.test.ts`:
the `coreEvents` import and two mock methods (`on`, `emit`) on the
hoisted `mockCoreEvents`. Sells itself as a "production readiness
review" but the actual change is a test-file cleanup of dead imports
and unused mocks.

## Specific citations
- `packages/core/src/config/config.test.ts:26` — drops
  `import { coreEvents } from '../utils/events.js';`. If nothing else
  in the file references `coreEvents`, this is correct dead-import
  removal. (The diff doesn't show any remaining reference, which is
  the implied premise.)
- `packages/core/src/config/config.test.ts:204-205` — drops
  `on: vi.fn()` and `emit: vi.fn()` from the `mockCoreEvents` object
  literal. Also dead per the PR claim.

## Concerns
1. **Title mismatch.** "Production readiness review: fix unused
   imports and security test coverage" overpromises. Nothing in the
   diff touches security tests or production behavior. Title should
   read something like `chore(test): drop dead coreEvents imports
   from config.test.ts`.
2. **No verification in diff.** The PR doesn't show `grep` evidence
   that `coreEvents`, `mockCoreEvents.on`, and `mockCoreEvents.emit`
   are actually unreferenced after the removal. A maintainer should
   confirm before merge — if any test in the file does
   `mockCoreEvents.on(...)` indirectly, this breaks.
3. **Author appears to be doing rapid janitorial PRs.** Worth
   checking whether this is part of an LLM-generated drive-by series
   (the "production readiness" framing is a red flag for that
   pattern).

## Verdict
**request-changes**

The change itself is probably fine, but the misleading title and
absence of any actual "security test coverage" delta (despite that
being half the title) suggest the PR description should be rewritten
before merge. Ask the author to (a) retitle to match the diff and
(b) confirm with `rg coreEvents packages/core/src/config/` that
nothing else in the file references the removed imports/mocks.
