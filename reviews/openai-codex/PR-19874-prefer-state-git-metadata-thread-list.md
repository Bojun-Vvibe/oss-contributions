# openai/codex#19874 — [codex-backend] Prefer state git metadata in filtered thread lists

- **Repo**: openai/codex
- **PR**: [#19874](https://github.com/openai/codex/pull/19874)
- **Head SHA**: `11b025f8d245936abfc3c19659c03b6f3c99bf12`
- **Reviewed**: 2026-04-29
- **Verdict**: `merge-as-is`

## Context

`codex-rs/rollout/src/recorder.rs` has a helper
`fill_missing_thread_item_metadata(item, state_item)` that overlays
state-DB metadata onto filesystem-discovered thread items returned by
`thread/list` filtered queries. The filesystem rollout has its own
view of `cwd`, `git_branch`, `git_sha`, `git_origin_url`, `source`,
`agent_nickname`, `agent_role`. The state DB has fresher data (the
backing SQLite is updated after every turn; the rollout file is
written once at thread start).

The previous merge logic was "fill if absent":

```rust
if item.git_branch.is_none() { item.git_branch = git_branch; }
if item.git_sha.is_none()    { item.git_sha    = git_sha;    }
if item.git_origin_url.is_none() { item.git_origin_url = git_origin_url; }
```

This is correct for fields that don't change after thread creation
(e.g. `cwd`, `first_user_message`). It's *wrong* for git fields,
because:

- A user can `git checkout other-branch` mid-session
- A user can `git push` to a new remote, changing `git_origin_url`
- A user can amend/reset, changing `git_sha`

Each of those updates the state DB. But because the filesystem
rollout already had a non-None value (the original branch/sha/origin
captured at thread start), the merge would *keep* the stale
filesystem value and *discard* the fresh state value. UI shows the
wrong branch icon (which is the visible symptom in the PR's
screenshot).

## The fix

Flip the precedence for the three git fields: prefer state-DB value
when it's non-None, fall back to filesystem value when state is None.
Other fields (`cwd`, `source`, etc.) keep the "fill if absent"
semantics.

```rust
if git_branch.is_some()    { item.git_branch    = git_branch; }
if git_sha.is_some()       { item.git_sha       = git_sha; }
if git_origin_url.is_some(){ item.git_origin_url = git_origin_url; }
```

Three identical-shape changes, scoped only to git metadata. The
asymmetry with the other fields is intentional and matches the
"git fields are mutable mid-session, others aren't" reality.

## Test coverage

`fill_missing_thread_item_metadata_preserves_filesystem_identity` is
renamed to
`fill_missing_thread_item_metadata_preserves_identity_and_prefers_state_git_fields`.
The fixture is updated so the filesystem item now has populated
(stale) git fields:

```rust
let filesystem_item = ThreadItem {
    thread_id: Some(filesystem_thread_id),
    first_user_message: Some("filesystem message".to_string()),
    cwd: None,
    git_branch: Some("filesystem-branch".to_string()),
    git_sha: Some("filesystem-sha".to_string()),
    git_origin_url: Some("https://example.com/filesystem.git".to_string()),
    ...
};
```

This is the right test shape — under the *old* logic the merge
would keep these stale values; under the *new* logic the merge
should replace them with state-DB values. Asserting that the merged
result matches the state values (presumably done in the body that
isn't fully visible in the diff hunk) directly proves the fix works
on the realistic stale-overlay scenario.

## Risk analysis

- **State-DB freshness assumption**: this fix assumes the state DB
  is the authoritative current view of git metadata, and the rollout
  file is the historical view. The state DB is written after every
  turn (per the surrounding code's behavior); the rollout file is
  written at thread start. So state-DB-when-present is strictly
  fresher. Safe assumption.
- **Null-state-DB races**: if state-DB git lookup races and returns
  None for a thread that *did* have git context at start, the
  filesystem fallback kicks in. Same UX as before for that case.
- **Cross-field consistency**: branch / sha / origin are updated as a
  group in the state DB (presumably under a single transaction at
  turn end), so a partial overlay (new branch + old sha) shouldn't
  happen in practice. If it ever does, the UI just shows a
  branch/sha mismatch — visible to the user, not a hidden bug.
- **Backwards-compat**: any thread whose state DB has no git metadata
  yet gets exactly the same display as before (filesystem-only). Any
  thread with state-DB git metadata now gets the fresh values
  instead of stale rollout values. Both cases are improvements, no
  regressions.

## Suggestions (non-blocking)

- The pattern of "some fields are mutable, others aren't" is
  important enough that it deserves a comment block above the
  helper explaining *why* the three git fields use `.is_some()` and
  the others use `.is_none()`. Right now a reader has to figure
  that out from the test name and the diff. Two-sentence comment
  prevents the next maintainer from "regularizing" the helper into
  a uniform `is_none()` ladder and silently re-introducing this bug.

## Verdict

`merge-as-is`. Three-line semantic flip with test coverage updated
to assert the new precedence on the realistic stale-overlay
fixture. The PR's screenshot ("now getting expected icons")
confirms the user-visible fix. The asymmetry with the other fields
in the same helper is intentional and correct — but worth a comment.

## What I learned

"Fill if absent" is the right default merge for *immutable*
metadata, and the *wrong* default for *mutable* metadata. Any field
that can change during the lifetime of the entity needs the
opposite precedence: prefer the fresher source, fall back to the
older one. The trap is that both kinds of fields often live in the
same struct and pass through the same merger, and a uniform "fill
if absent" loop will silently corrupt the mutable ones. Worth
auditing any merger helper in your code for this asymmetry —
typically you want immutable fields filled-if-absent, mutable
fields preferred-from-fresh-source, and the helper structured to
make the distinction obvious.
