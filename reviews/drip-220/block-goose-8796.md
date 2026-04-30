---
pr-url: https://github.com/block/goose/pull/8796
sha: f827af6295ec
verdict: merge-as-is
---

# fix: use _meta instead of meta in newSession request

Two-line correctness fix at `ui/goose2/src/shared/api/acpApi.ts:189-199` swapping the `meta` field name to `_meta` in the `newSession` ACP request, plus removing the `& { meta?: Record<string, string> }` ad-hoc type intersection that had been widening the `Parameters<typeof client.newSession>[0]` shape to permit the wrong field name.

The bug shape is excellent: ACP's wire protocol uses `_meta` (underscore-prefixed) for caller-supplied opaque metadata — the underscore is significant because it signals "this field is opaque to the protocol layer; pass through unchanged" — and the prior code emitted `meta` (no underscore), which the backend silently dropped. The user-visible symptom per the PR body was sessions not being associated with their projects: the `projectId` value was being assembled into the `meta` dict at `:194-196` and then attached to `request.meta` at `:199`, but the backend's session-create handler reads `request._meta.projectId`, found nothing, and created a project-less session.

The deletion of the type intersection is the load-bearing structural improvement: by widening the request type with `& { meta?: Record<string, string> }`, the original code was *defeating the very type-checker that would have caught this bug*. With the widening removed, the new `request._meta = meta` line at `:199` typechecks against the canonical `client.newSession` signature, and any future protocol-field-name typo will fail at compile time instead of silently shipping. This is the textbook case of "don't add type intersections to silence a type error — the type error was telling you something."

The PR has zero test additions which is mildly disappointing for a correctness fix, but the change is so localized (one renamed key, one deleted type assertion) that a unit test pinning "we sent `_meta` not `meta`" would essentially restate the diff. A higher-leverage test would be at the integration layer — round-trip a `projectId` through `newSession` and assert the returned session's `projectId` matches — but that's a separate gap to file rather than blocking this fix.

## what I learned
Type-system escape hatches (`as`, `&`, `unknown`-then-cast) accumulate technical debt in the worst possible shape: they're invisible at use sites, they suppress the exact kind of error the type checker exists to catch, and removing them later requires reproving the safety claim that was never written down. Whenever you find yourself adding `& { ... }` to widen a generated/imported type, the right next step is "open the source of the original type and either (a) update the source to include the new field as a real contract, or (b) figure out why your shape disagrees with the canonical one and fix yours." Choice (c), "widen with an intersection and ship," guarantees this exact bug class within ~3 months.
