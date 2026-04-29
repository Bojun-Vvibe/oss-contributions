# openai/codex#20095 — `permissions: expose active profile metadata`

- PR: https://github.com/openai/codex/pull/20095
- Head SHA: `b56d4132b00bb4dc36b25e509af60f058de62c50`
- Author: bolinfest

## Summary of change

Adds two new wire-protocol types — `ActivePermissionProfile` and
`ActivePermissionProfileModification` — to the v2 app-server
protocol, plus matching `PermissionProfileSelectionParams` /
`PermissionProfileModificationParams` request shapes. Replaces the
previous `permission_profile: PermissionProfile` field on thread
start/resume responses with a pair: `permission_profile` (now
nullable) and `active_permission_profile` (the new shape, carrying
profile `id`, optional `extends`, and a list of bounded
`modifications`). `ThreadForkParams` is rewritten to use the new
modification shape instead of the old `FileSystemAccessMode` /
`FileSystemPath` types.

## Specific observations

- `codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.v2.schemas.json:130-189` —
  `ActivePermissionProfile.extends` is declared as `["string", "null"]`
  with `default: null` and the comment "currently always `null`".
  Shipping a field as part of the wire contract that you intend to
  use later is exactly how you end up with two clients reading
  `extends` differently in three months. Either gate it behind a
  feature flag in the schema or add a `"deprecated": false` +
  changelog note explicitly stating today's invariant.
- The `modifications` array currently has only one variant
  (`additionalWritableRoot`). The `oneOf` shape is correct
  forward-design — adding `removeReadableRoot` etc. later is purely
  additive. But the runtime needs an explicit "unknown variant ⇒
  reject" path on the *server* side (not visible in this diff) so a
  malicious client can't smuggle through a future variant string
  that an old server happily ignores. Worth a paired core-side check.
- `codex-rs/analytics/src/analytics_client_tests.rs:65-216` — the
  test fixture `sample_thread_start_response` /
  `sample_thread_resume_response_with_source` now sets both
  `permission_profile: None` and `active_permission_profile: None`.
  The previous test was actually exercising the `Some(...)` branch
  via `sample_permission_profile()` — and that helper is removed
  (line 169-172 of the diff). After this PR there is *zero* test
  coverage of the populated `active_permission_profile` shape on
  the wire. At least one fixture should be flipped to a populated
  `ActivePermissionProfile { id: ":workspace", extends: None,
  modifications: vec![...] }` so JSON round-trip is covered.
- The three `ThreadStartResponse` / `ThreadResumeResponse` /
  `ThreadForkResponse` variants in the v1 schema all get the same
  description softening: "New clients should use `permissionProfile`
  when present" → "Experimental clients should prefer
  `permissionProfile` when they need exact runtime permissions."
  This is a *weakening* of the previous guidance — clients that read
  the previous wording and switched to `permissionProfile` are now
  being told it's experimental. If the v1 stance is "stay on legacy
  `sandbox`," this is a contract drift worth a CHANGELOG entry, not
  just a description tweak.
- `ThreadForkParams.json:64-128` — the old
  `FileSystemAccessMode` (`read`/`write`/`none`) and `FileSystemPath`
  (`path` | `glob_pattern`) types are *deleted* from the schema.
  This is a breaking change for any client that constructed a fork
  request against the old shape. The PR title says "expose active
  profile metadata" — this rename/replace is hidden under that title
  and deserves its own callout in the description.

## Verdict

`request-changes` — protocol-level surgery with a forward-leaning
`extends` placeholder, a silent test-coverage regression on the
populated-profile path, a hidden breaking change to
`ThreadForkParams`, and the weakening of v1 client guidance from
"should use" to "experimental". Each of those wants explicit
acknowledgement before this lands on a stable wire.
