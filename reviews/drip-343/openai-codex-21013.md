# openai/codex #21013 — fix(plugins) install/update flow reliability

- PR: https://github.com/openai/codex/pull/21013
- Head SHA: `5dc522afbe48291299ac7630a48d51ee7572631a`
- Diff size: +170 / -0, 1 file (test-only)

## Summary

Adds a Windows-gated regression test
(`codex-rs/app-server/tests/suite/v2/plugin_install.rs`) named
`plugin_install_upgrades_while_plugin_mcp_server_holds_old_version_cwd`
that exercises the case where a plugin install/upgrade is requested
while a stdio MCP server from the previous version still holds its
working directory open. On Windows this is the classic "file in use"
/ "Access is denied" failure mode for `rename`/`remove_dir_all`.

## Citations

- `plugin_install.rs:11-12, 31-32` — new `#[cfg(windows)]` imports
  for `create_mock_responses_server_repeating_assistant` and
  `ThreadStartParams`. Cleanly gated; non-Windows compilation is
  unaffected.
- `plugin_install.rs:1086-1170` — test scaffolding: writes config
  with sample-plugin@debug, installs v1.0.0, registers v2.0.0 in the
  marketplace, starts a thread, waits for the stdio MCP "ready"
  notification, then triggers an upgrade.
- The test is cfg-gated to Windows but the underlying handler being
  exercised is cross-platform. That's intentional (only Windows
  exhibits the bug) but worth flagging: a regression that fires on
  Linux/macOS due to e.g. an `fs::rename` becoming a
  not-atomic-on-Windows op would not be caught here. Consider a
  parallel test that drops the `#[cfg(windows)]` and just asserts
  the upgrade succeeds with an open MCP — useful as a smoke test
  even on Linux.
- I did not see a corresponding code change in this diff — only the
  test. Either the prod fix is in a separate commit on this branch
  (verify against `5dc522af...`'s parents) or this PR depends on a
  prior change. Reviewer should confirm the upstream fix exists,
  otherwise this test will be red on Windows CI.

## Risk

Test-only diff. Risk of CI flake on Windows runners if the MCP
"ready" notification has timing variance — the test relies on
`mcp.read_stream_until_matching_notification` with `DEFAULT_TIMEOUT`
which should be generous, but Windows file locking can be slow.

## Verdict

`needs-discussion` — the diff as presented is test-only with no
visible production change in the same PR. Need to confirm the fix
itself lands here (or as a dependency) before approving; the test
shape is otherwise fine.
