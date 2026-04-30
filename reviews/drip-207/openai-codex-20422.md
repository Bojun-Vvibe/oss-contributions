# openai/codex #20422 — mcp: send sandbox metadata as permission profile only

- Head SHA: `4a90f762986121c66074a1117548b96f062cf50f`
- Files: `codex-rs/codex-mcp/src/runtime.rs`, `codex-rs/core/src/mcp_tool_call.rs`, `codex-rs/core/tests/suite/rmcp_client.rs`
- Size: +3 / -8

## What it does

`SandboxState` (the metadata blob shipped to MCP servers under
`codex/sandbox-state-meta`) carried both `permission_profile: Option<PermissionProfile>`
and a duplicated `sandbox_policy: SandboxPolicy` field — the latter being a
legacy, lossy projection of the former (it cannot represent split filesystem
rules like deny-read carve-outs over an otherwise-readable workspace, bounded
glob scans, or other non-trivial profile shapes).

This PR:

- Drops the `sandbox_policy: SandboxPolicy` field from the `SandboxState`
  struct in `codex-rs/codex-mcp/src/runtime.rs:23`, and removes the
  `use codex_protocol::protocol::SandboxPolicy;` import at line 14.
- Stops constructing it at the call site in
  `codex-rs/core/src/mcp_tool_call.rs:580` (one line removed inside
  `serde_json::to_value(SandboxState { … })`).
- Updates the integration test at
  `codex-rs/core/tests/suite/rmcp_client.rs:824-828` to assert
  `sandbox_meta.get("permissionProfile") == Some(profile_json)` and
  `sandbox_meta.get("sandboxPolicy") == None` (i.e. the legacy field is
  gone from the wire, not just renamed).
- Removes the now-unused `turn_permission_fields(...)` invocation that was
  there only to compute the legacy projection.

## What works

This is the right cleanup. The migration to `PermissionProfile` as the
canonical permission model has been a multi-PR effort across the repo (see
the long bolinfest series #20387–#20421 in the same batch). With `SandboxState`
carrying both fields, there was a real risk of MCP server authors keying on
`sandboxPolicy` because that was the older, more documented name — and then
silently getting wrong behavior for any thread using profile features that
don't downgrade cleanly. Removing it now, while the v1 wire is still
warmable, is cheaper than removing it after the next release.

The test update is the right shape: it asserts the *positive* presence of
`permissionProfile` AND the *absence* of `sandboxPolicy`. That second
assertion is what catches a future regression where someone re-adds the
field "for compatibility".

## Concerns

1. **Wire-version signaling.** This is a breaking change to the
   `codex/sandbox-state-meta` payload, but I don't see a corresponding
   bump or note in the MCP protocol docs. Any external MCP server that
   was reading `sandboxPolicy` will silently get `None` after this lands.
   A line in CHANGELOG / a bump of whatever capability versioning the
   sandbox-state meta has would be appropriate. If there is no versioning
   today, this is a good moment to add a `version: 2` field in the same
   diff.
2. **Migration window.** The PR description argues no first-party MCP
   server still depends on the legacy field. That's likely true but
   unverified for third-party servers. Consider keeping the field
   serialized (with a deprecation warning logged once per process) for
   one release, then removing it. Three lines of diff become four;
   blast radius drops to zero.
3. **No regression test for the rich-profile case.** Since the entire
   point of the migration is that `PermissionProfile` carries shape that
   `SandboxPolicy` couldn't, the new assertion at
   `rmcp_client.rs:824-828` should ideally exercise a profile with a
   deny-read carve-out (or similar non-projectable rule) and assert the
   MCP server-visible JSON preserves it. As written, the test uses
   `PermissionProfile::read_only()` which round-trips fine through the
   legacy shape too — so it doesn't actually demonstrate the lossy-ness
   the PR description cites as motivation.

Verdict: merge-after-nits

## What I learned

When killing a "compat" field that was explicitly added as a lossy
projection of a richer one, the cheapest insurance is a one-release
deprecation window with a per-process log warning. It costs almost
nothing and turns a silent breakage for downstream MCP servers into
a noisy one. Also: when the test uses the simplest profile that
round-trips through both shapes (`read_only()`), it doesn't actually
exercise the property that motivated the change. Worth picking a test
fixture that would have *failed* under the legacy field.
