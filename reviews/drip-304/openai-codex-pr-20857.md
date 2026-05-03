# openai/codex #20857 — Add vanilla context mode

- **PR:** https://github.com/openai/codex/pull/20857
- **Head SHA:** `8d96b25122cf181e84d617ca78841c37efa0e942`
- **Author:** gustavz
- **Size:** +457 / -65 across multiple files (protocol schemas + core)

## Summary

Introduces a `ContextMode` enum (`default | vanilla`) plumbed through `ThreadStartParams`, `ThreadResumeParams`, and `ThreadForkParams`. Vanilla mode suppresses configured instructions, AGENTS.md, skills, plugin instructions, environment context, and personality — useful for benchmark / diff debugging where you want the raw model behavior without the harness's context layering.

## Specific references

- `codex-rs/app-server-protocol/schema/json/ClientRequest.json:540-546` — new `"ContextMode"` definition with enum values `["default","vanilla"]`. Plain string enum (not tagged), matches sibling enum styles in the schema.
- Same file, hunks at `:3464`, `:3878`, `:4066` — `contextMode` field added to all three thread params via `anyOf: [$ref ContextMode, null]`. Nullable shape is correct for backward-compat (existing clients that don't send the field get null → default behavior).
- TypeScript schema mirrors at `schema/typescript/v2/{ContextMode,ThreadForkParams,ThreadResumeParams,ThreadStartParams}.ts`.
- PR body mentions app-server resume mismatch reporting is covered by tests; thread fork/resume must reject a `vanilla` resume on a `default`-mode parent (or vice versa) — that's the mismatch case worth verifying in review.

## Concerns

1. **Resume/fork compatibility semantics** — when resuming a thread, what happens if the caller passes a `contextMode` that disagrees with the thread's original mode? The body says "resume mismatch reporting covered by tests" — reviewers should confirm the test asserts an error rather than silently switching mode mid-thread (which would be confusing).
2. **Two schema files (`v1` + `v2`) updated** — make sure both stay in lockstep. The PR appears to do this (`codex_app_server_protocol.schemas.json` and `.v2.schemas.json` both touched).
3. **AGENTS.md suppression in vanilla** — this is a meaningful behavior change. A user running `codex` in a directory with a project-specific AGENTS.md who flips to vanilla mode could be confused that their project rules disappeared. Worth a CLI hint or doc note.

## Verdict

**needs-discussion** — the feature is well-scoped and the protocol shape is clean, but the semantics around resume-with-mismatched-mode and the UX implication of suppressing AGENTS.md deserve maintainer sign-off before merge. Not a request-changes — the implementation looks fine — but a maintainer should confirm intent.
