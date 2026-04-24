# openai/codex PR #19236 — Add instruction params to app-server-test-client

Link: https://github.com/openai/codex/pull/19236
State: OPEN · Author: pakrym-oai · +116 / −17 · Files: 1
(`codex-rs/app-server-test-client/src/lib.rs`)

## Summary
Threads two new global CLI flags — `--base-instructions` and `--developer-instructions` — through
every subcommand of the app-server test client and applies them to both `ThreadStartParams` and
`ThreadResumeParams`. A new `ThreadInstructionOverrides` helper centralizes the mapping. The
goal is to let test runs override base/developer instructions without rebuilding fixtures.

## Strengths
- The `ThreadInstructionOverrides` newtype with `apply_to_thread_start` / `apply_to_thread_resume`
  is a clean pattern — overrides are built once and applied at both spawn sites without
  duplicated assignment code.
- All 9-ish subcommands consistently take `&instruction_overrides`, so the contract is uniform
  across commands; nobody has a stale signature.
- Globals with `global = true` mean overrides work both pre- and post-subcommand on the CLI,
  which matches user muscle memory from other clap-based tools.

## Concerns / Risks
- **Unconditional clobber: `apply_to_thread_*` always assigns `self.X.clone()` to the params
  fields, even when the override is `None`.** That means callers who themselves populate
  `ThreadStartParams.base_instructions` (e.g., a future code path that derives base instructions
  from a config file) will have those values silently overwritten with `None` by this helper.
  The right semantics is `if let Some(v) = &self.base_instructions { params.base_instructions =
  Some(v.clone()); }` — leave the field alone when no override is present. This is a textbook
  "merge vs. clobber" edge case and will bite the next person who chains config sources.
- **No validation on the `text` value.** `value_name = "text"` accepts arbitrary strings,
  including embedded NULs, ANSI escapes, or multi-MB blobs piped via `--base-instructions
  "$(cat huge.md)"`. The server-side instruction parsing has its own limits, but a missing
  client-side cap means errors surface as opaque deserialization or transport failures rather
  than a clear "instructions too large" message.
- **No `@file` syntax**, unlike the sibling `--dynamic-tools` flag, which already accepts
  `json-or-@file`. For instructions of any non-trivial length, users will hit shell argv
  limits quickly. Mixing conventions across flags in the same binary is a small but real UX
  papercut.
- **No tests in this PR.** The diff is +116/−17 of test-client wiring with zero regression
  coverage that the new params actually flow into outbound JSON-RPC requests. A snapshot test
  asserting that `--base-instructions foo` produces `"base_instructions":"foo"` on the wire
  would catch a future field-name rename in `ThreadStartParams`.
- **Send-message v1 path** (`CliCommand::SendMessage`) routes through
  `send_message_v2_with_policies`, but the public `send_message_v2` entrypoint defaults
  `instruction_overrides` to `Default::default()`, so external callers of the library
  function still cannot pass overrides — only the binary can. Worth aligning the library and
  CLI surfaces.

## Suggested follow-ups
1. Switch `apply_to_thread_*` to merge semantics so `None` overrides do not clobber existing
   values.
2. Add `@file` support to both new flags for symmetry with `--dynamic-tools` and to dodge
   argv-size limits on long instructions.
3. Add at least one snapshot test asserting the JSON-RPC payload contains the override fields
   when set and omits them when unset.
