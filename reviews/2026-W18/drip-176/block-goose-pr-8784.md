# block/goose#8784 — fix: use python3 in developer extension instructions for macOS/Linux compatibility

- PR: https://github.com/block/goose/pull/8784
- Head SHA: `07572e7bfa58dd9afe1fce6bfa41eecc2733cbff`
- Author: treebird7 (Treebird)
- Base: `main`
- Size: +6 / -0 across 2 files (1 source + 1 snapshot)

## Summary

Adds two lines to the non-Windows developer-extension system prompt
instructing the agent to "always use `python3` instead of `python`."
Modern macOS (12+) shipped without `python` (only `python3`), and many
Linux distros (Debian/Ubuntu since 20.04 default) follow the same
convention. The change is inside the existing `cfg!(windows)` else
branch at `crates/goose/src/agents/platform_extensions/developer/mod.rs:60-62`,
so Windows behavior is unaffected. Snapshot file
`__all_platform_extensions.snap` is updated in lockstep.

## Notable design choices

- The instruction is added at the *end* of `developer_instructions()`
  for the non-Windows branch, after the existing rg/cat/sed/edit
  guidance. This placement reads naturally as "additionally, when you
  use Python..." Good.
- `mod.rs:60-61` wording: "When running Python scripts or commands,
  always use `python3` instead of `python`. The `python` command is
  not available on modern macOS and many Linux distributions." Two
  sentences. Imperative + justification. Concise.
- Snapshot update at
  `goose__agents__prompt_manager__tests__all_platform_extensions.snap:93-94`
  is exactly the two new lines, no other drift. The existing snapshot
  test (which the PR doesn't add a new case for) provides automatic
  protection — if someone removes the `python3` line in a future
  refactor, `cargo insta` will flag it.

## Concerns

1. **PR body says "No existing tests cover the content of
   `developer_instructions()` — tests in mod.rs are functional
   tool-call tests."** This is half-true: the *snapshot* test (which
   was updated) effectively *does* cover the content, just at the
   "all_platform_extensions" aggregation level rather than per
   function. Author should update the PR body to acknowledge the
   snapshot is the regression guard — otherwise the next contributor
   might think there's no test coverage and drop the line.
2. **The fix is documentation, not enforcement.** The agent can
   still ignore the instruction and emit `python` — the only thing
   that's changed is the prompt nudge. A more durable fix would be
   to:
   - intercept tool-call shell commands and rewrite `^python\s` →
     `python3 ` at the executor layer
   - or add a pre-flight check that warns when the agent emits
     `python` on a system where `which python` returns no result

   Neither is in scope for a one-line prompt fix, but worth noting
   the limitation in the PR body.
3. **Linux coverage is over-stated.** "Many Linux distros" varies:
   RHEL/CentOS 8+ requires explicit `dnf install python3` and
   doesn't symlink either way; Arch defaults to `python` →
   `python3`; Alpine ships `python3` only by default; older Ubuntu
   16.04/18.04 still have `python` → `python2`. The guidance
   "always use `python3`" is correct *as a defensive default* but
   will produce a confusing failure on systems where `python3` is
   *also* missing (Alpine without the python3 package). Consider
   rewording to "prefer `python3` over `python` — `python` is not
   available on modern macOS or Debian-based Linux 20.04+."
4. **Wording "not available on modern macOS" at `mod.rs:61`.** This
   is an absolute claim. Users with Homebrew-installed `python` or
   pyenv-managed `python` *do* have a `python` binary. The
   instruction still produces the right behavior (use `python3`,
   which also exists in those setups) but the justification is
   shaky. "Often unavailable" or "not guaranteed to be present"
   would be more accurate.
5. **No companion change for related tools.** If the agent is
   advised to use `python3`, it should also use `pip3` not `pip`,
   `python3 -m venv` not `virtualenv`, etc. Out of scope here, but
   filing a follow-up issue to scrub the prompt for similar bare-
   binary references would close the loop.

## Verdict

**merge-after-nits** — correct fix for a real bug (#8756 is a known
class of agent failures: model emits `python script.py`, tool fails,
agent retries with confused error). The wording in items 3-4 (Linux
coverage and absolute "not available" claim) is the only thing
worth tightening before merge. The snapshot update keeps the
regression risk low. Ship after one round of wording polish.
