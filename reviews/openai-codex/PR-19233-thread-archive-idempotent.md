# PR #19233 — make `thread/archive` idempotent for already-archived threads

**Repo:** openai/codex • **Author:** guinness-oai • **Status:** closed
(superseded) • **Net:** +84 / −1

## Change

Single-line change in `codex_message_processor.rs`:

```diff
-        include_archived: false,
+        include_archived: true,
```

inside the `thread/archive` handler's parent-thread read, mirroring
what the descendant path already did. Plus an 83-line regression test
`thread_archive_is_idempotent_for_already_archived_thread` that
archives, asserts the rollout moved to the archived sessions
subdirectory, archives again, and asserts the archived rollout path
is unchanged.

## What was actually broken

The PR description names it: stale UI state. The app caches a thread
as "active" in the sidebar; the server has already archived it (via
another window, another device, or a TTL sweep). The user clicks
"Archive" again, the server tries to load the parent rollout from
the active path only, fails with `no rollout found for thread id`,
and the UI shows "Failed to archive conversation" with a flicker.

The author correctly distinguishes the symptom (`Worktree cleaned
up` banner co-occurring) from the cause (server-side active-only
read). The cleanup path runs only on a successful first archive, so
its banner being present means a *prior* archive succeeded — exactly
the "already archived" precondition.

## Why a server-side fix and not just UI invariant enforcement

The author flags this explicitly: "visible active threads should
never be archived" *is* a UI invariant worth enforcing, but the UI
state can leak (multi-window, multi-device, push-notification
delivery delays). Making the server idempotent is the correct
backstop because:

1. It's a one-line change.
2. It preserves the non-success cases that *should* fail
   (`thread_archive_requires_materialized_rollout` — archiving a
   thread that never produced a rollout still errors).
3. It removes a class of "ghost failure" support tickets that have
   no good user-side mitigation.

## Risk: read-after-archive semantics

`include_archived: true` causes the parent read to find the rollout
in the archived subdirectory. The handler then proceeds to its
archive logic. The test asserts the **path is unchanged** after the
second archive — meaning the second archive call finds the already-
archived rollout, recognizes there's nothing to move, and returns
success. What the test does *not* assert is whether
`thread/archived` notification is re-emitted on the second call.

Re-emitting it is the conservative choice — clients that missed the
first notification get a second chance. Suppressing it avoids
spurious UI animations on idempotent calls. The PR description
doesn't say which path is taken; reading the diff, the handler runs
to completion either way, so a second notification is likely
emitted. Worth a regression assertion in the test:
`assert!(no_second_archived_notification)` or
`assert!(second_archived_notification_is_emitted)` — pin one of
them so future refactors don't quietly flip the contract.

## Sharp edge: descendant handling

The handler already did `include_archived: true` for descendants —
that's how this PR was discovered (the asymmetry was the smell). The
descendant path likely already handles "already archived" gracefully.
But the *combined* case — parent already archived, some descendants
active — is now reachable. Test should add: archive a parent +
spawn a new descendant after archive (or via a stale UI), then call
`thread/archive` on the parent again, and assert the descendant is
also moved without error. This catches the case where the parent is
a no-op but a descendant still has work to do.

## Verdict

Correct, minimal, well-justified. The 83 lines of test for one line
of fix is appropriate — the failure mode is subtle and the
regression cost would be high. Closed (superseded), presumably
folded into a broader cleanup, but the analysis stands as a model
for "small server backstop > heavyweight UI invariant" trade-offs.

## What I learned

When two paths in a handler use different read-mode flags
(`include_archived: false` vs `true`), that asymmetry is almost
always a bug. The descendant path got it right first; the parent
path is the cleanup. Worth grepping any other `include_archived:
false` sites for the same smell.
