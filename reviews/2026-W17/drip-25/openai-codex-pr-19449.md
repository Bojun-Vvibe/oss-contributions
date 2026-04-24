# openai/codex PR #19449 — permissions: remove legacy read-only access modes

- **Repo:** openai/codex
- **PR:** [#19449](https://github.com/openai/codex/pull/19449)
- **Head SHA:** `bbc6c1beb896668f2243ed65a1230e4b9b44eb97`
- **Author:** bolinfest (Michael Bolin)
- **Size:** +153 / -1492 across 63 files
- **Reviewer:** Bojun (drip-25)

## Summary

Sixth PR in the `PermissionProfile` migration arc. Drops the
transitional `ReadOnlyAccess` enum from
`codex_protocol::protocol::SandboxPolicy`, so the legacy projection
no longer carries split-readable-roots / deny-read-globs / platform-
default-read state. That state now lives entirely in
`FileSystemSandboxPolicy` and `PermissionProfile`, where the
permissions migration has been pushing it for the past month.

`SandboxPolicy::ReadOnly` and `SandboxPolicy::WorkspaceWrite` revert
to representing the *historical full-read* sandbox modes only — i.e.
the legacy enum stops trying to be expressive and goes back to being
a strict back-compat shim.

## Key changes

### `codex-rs/protocol/src/protocol.rs` (+3 / −244)

The biggest single deletion. `ReadOnlyAccess` and its `access` /
`readOnlyAccess` API fields are gone from the wire enum. The
serde-tolerance commitment stays explicit in the PR body:

> Preserves compatibility for older app-server clients by
> continuing to deserialize payloads that include the removed
> `access` or `readOnlyAccess` fields and ignoring those fields.

Worth pinning that contract in a regression test (a JSON fixture
with the old field present, asserting it round-trips through deserialize +
re-serialize without panicking and without re-emitting `access`). I
don't see one in the visible diff — the test deletions are mostly
removing now-impossible `Restricted` arms from match expressions,
not adding new ignore-unknown-field tests.

### `codex-rs/protocol/src/permissions.rs` (+45 / −147)

Conversion paths are simplified. The previous "unmap restricted
read roots back into the legacy enum" code goes away entirely
(legacy modes no longer have a slot to put them in). The +45/-147
delta is the right shape: removing dead bookkeeping, not papering
over it.

### `codex-rs/sandboxing/src/policy_transforms.rs` (+2 / −36)

`ReadOnlyAccess::Restricted` projection paths drop out. The
remaining `+2` is the explicit `unwrap_or` for the
`platform-default-read` flag — which used to come implicitly from
the enum tag and now needs to be passed in.

### `codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs` (+7 / −151)

Big deletion. The Windows sandbox previously carried its own copy
of "what counts as restricted-readonly" logic so it could honour
the `ReadOnlyAccess::Restricted` distinction. That logic moves
out of this file because the runtime now passes an explicit
override flag (`identity.rs` +3, `elevated_impl.rs` +3 / −1).
This is the cleanup I'd most want documentation on — the runtime
contract for the new override flag isn't reflected in any new
doctest I can see.

### `codex-rs/sandboxing/src/restricted_read_only_platform_defaults.sbpl` (+1 / −1)

A one-line seatbelt profile bump, presumably to match the new
default-read shape. Worth confirming with a `seatbelt_tests.rs`
review that the resulting profile still denies the same paths it
denied before — the `seatbelt_tests.rs` delta in this PR is
`-46` (pure deletion of `Restricted` test cases), so the new
default-read coverage relies on whatever was already in place for
`ReadOnly` / `WorkspaceWrite`.

## Concerns

1. **Old-client back-compat is asserted in prose, not in a test.**
   The PR description promises that older app-server clients
   sending `access` or `readOnlyAccess` will still deserialize.
   Given the `-67` delta repeated across 8 separate JSON schema
   files (4 v2 schemas + 4 root schemas), the next maintainer who
   touches `serde(deny_unknown_fields)` could silently break this.
   Ask: a small fixture-driven test in
   `codex-rs/protocol/tests/legacy_compat.rs` with a checked-in
   pre-PR JSON payload that asserts deserialize succeeds and
   re-serialize doesn't emit the dropped fields.

2. **Windows override flag isn't documented at the call site.**
   The 151-line deletion in `setup_orchestrator.rs` is replaced
   by a small flag passed in from the runtime. The flag's intent
   ("this profile wants the platform-default minimal-read set
   instead of full-read") isn't obvious from its name alone (I
   can't see the exact name in the available diff snippet, but
   `identity.rs +3` and `elevated_impl.rs +3 / −1` are where it
   lands). A two-line doc-comment on the field would prevent the
   next reader from mis-defaulting it.

3. **`seatbelt_tests.rs −46` is pure deletion.**
   Tests that exercised `Restricted` are removed because the arm
   doesn't exist anymore. Fine. But the *behavioural* coverage
   they provided (e.g. that `~/Library/Caches` still gets denied
   under restricted reads) needs to be re-asserted under the new
   `FileSystemSandboxPolicy` path. If those assertions already
   exist in the new permissions tests, a comment pointing
   reviewers there would close the audit loop.

4. **Stack of 6 PRs landing together raises rebase risk.**
   This is the head of a Sapling stack on top of #19395 / #19394 /
   #19393 / #19392 / #19391. If any of those re-spin, the conflict
   surface in `protocol.rs` (-244 lines is a lot of context) gets
   ugly fast. Worth landing the base first and rebasing this one
   on the merged tip before final review, rather than reviewing
   the stack interleaved.

## Verdict

`merge-after-nits` — direction is correct (legacy enum stops
pretending to carry information that now lives elsewhere; +153 /
−1492 is the deletion shape we want at this stage of the
migration). Two follow-ups before merge: a serde back-compat
fixture test for the dropped `access` / `readOnlyAccess` fields,
and a doc-comment on the new Windows platform-default-read flag.
The seatbelt test deletions are fine as long as the equivalent
assertions are reachable from the new `FileSystemSandboxPolicy`
test surface — quick cross-reference comment satisfies that.

## What I learned

The "transitional enum that carried partial information until the
real model was ready" pattern is a clean way to migrate a public
wire format incrementally — the legacy enum stays expressive
enough to round-trip during the migration, then collapses back to
its original shape once consumers have moved over. The cost is
that the collapse PR (this one) is the riskiest of the stack:
every consumer that learned to read the transitional fields needs
to be cleaned up in the same change, and the back-compat
contract for old senders needs to be pinned with an explicit test
because it stops being syntactically obvious. Same shape as the
`SandboxPolicy::DangerFullAccess` collapse two quarters ago, and
the LSP `WorkspaceFolders` capability dance — the lesson is to
ship the "remove the transitional shape" PR with a serde-tolerance
fixture, even when the prose promises tolerance.
