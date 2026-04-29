# openai/codex#20207 — fix(tui): make /copy work inside tmux without passthrough

- **PR**: https://github.com/openai/codex/pull/20207
- **Head SHA**: `f82610d1d6962cfc21972fab8eded8041ce6ab96`
- **Size**: +412 / −42, 1 file
- **Files**: `codex-rs/tui/src/clipboard_copy.rs`
- **Fixes**: #19926

## Context

`/copy` in the TUI was using DCS-wrapped OSC 52 whenever `TMUX` was set, which only reaches the outer terminal when tmux's `allow-passthrough` option is enabled. With passthrough off (the default in many distros and security-conscious setups), the TUI reported "copied" but nothing actually landed in the system clipboard. This PR adds tmux's *native* clipboard path (`tmux load-buffer -w -`) as the preferred mechanism inside tmux, with OSC 52 retained as fallback when tmux clipboard forwarding is misconfigured.

## Why this exists / problem solved

The original module-doc at `clipboard_copy.rs:5-11` only had two cases: SSH → OSC 52, local → arboard then WSL then OSC 52. Inside tmux specifically, OSC 52 is unreliable (passthrough required), but tmux *itself* knows how to load text into the system clipboard via `set-clipboard on` + a working `Ms` terminfo capability. The right answer is "ask tmux to do the copy" before falling back to OSC 52, and that's exactly what this PR adds.

## Design analysis

### The new selection ladder

The selection logic is restructured around a `CopyEnvironment { ssh_session, wsl_session, tmux_session }` struct at `clipboard_copy.rs:87-92`, replacing the old pair of `bool` parameters. This is a clean refactor — the old call shape would have grown to four positional bools (`copy_to_clipboard_with(text, ssh, wsl, tmux, ...)`), which is a known foot-gun. Wrapping them in a named struct also makes the test cases (e.g., `CopyEnvironment { ssh_session: true, wsl_session: false, tmux_session: true }`) self-documenting.

The new dispatch flow at `clipboard_copy.rs:103-159`:

1. **SSH session** → `terminal_clipboard_copy_with(text, tmux_session, ...)`. If tmux is also detected, try `tmux load-buffer -w -` first, fall back to OSC 52. This is the right call for the "SSH into a host that's running tmux" case which is extremely common.
2. **Local session, arboard succeeds** → use it.
3. **Local + WSL fallback fails** → terminal-mediated copy (tmux preferred, OSC 52 fallback).
4. **Local + non-WSL arboard fails** → terminal-mediated copy.

The error-message threading is careful — the WSL path at `:130-153` assembles a three-clause error string when terminal copy fails (`"native: ...; WSL: ...; terminal: ..."`), and the message branches on `tmux_session` so the user sees "tmux fallback failed" vs "OSC 52 fallback failed" depending on which one was actually tried. That's a small attention-to-detail win for support diagnosis.

### The `tmux_clipboard_copy_ready` precheck

At `clipboard_copy.rs:381-399` the PR adds a precheck that runs before `tmux load-buffer`: `tmux show-options -gv set-clipboard` must not return `"off"`, and `tmux info` must not show `"Ms: [missing]"`. This is the right gate — without `set-clipboard on` AND a working `Ms` capability, `tmux load-buffer -w` succeeds locally (the buffer is created) but tmux silently drops the forward-to-outer-terminal step, so the user gets a misleading success. The precheck converts that silent-success into an explicit fall-through to OSC 52.

Two subtleties on the precheck:

1. **`set-clipboard external` is treated as "on".** The check is `set_clipboard.trim() == "off"` — anything else (including the relatively common `external` value, which means "tmux forwards but doesn't keep its own buffer") passes. This is correct.
2. **The `Ms: [missing]` substring match at `:393`** is a `tmux info` text-format dependency. If tmux changes the rendering of missing capabilities (e.g., to `Ms (missing)` or `Ms = -`), the check will silently pass even when the capability is actually missing, and `tmux load-buffer` will return success without forwarding. This is a small dependency on tmux's `info` output format that should at least have a comment pinning the expected tmux version range.

### The `tmux_clipboard_copy` invocation itself

At `clipboard_copy.rs:319-377`, the PR spawns `tmux load-buffer -w -` with `stdin=piped, stdout=null, stderr=piped`, writes the bytes, drops stdin, and waits. This is textbook subprocess hygiene:

- `drop(stdin)` at `:351` correctly closes the write side before `wait_with_output()` so `tmux` sees EOF and exits.
- Failure paths at `:336-348` correctly `kill()` + `wait()` the child to avoid zombies if writing to stdin fails.
- The `stderr.is_empty()` check at `:367-374` falls back to "tmux exited with status N" instead of "tmux failed: " (empty string), so the error message is always informative.
- One concern: there's **no timeout** on `wait_with_output()`. If `tmux load-buffer` hangs (e.g., the tmux server is wedged or in the middle of a config reload), `/copy` hangs the TUI indefinitely. For a clipboard write this is probably fine in practice — `load-buffer` is a millisecond operation — but adding a small `wait_timeout` (1-2 seconds) would harden against pathological cases.

### Test coverage

The test imports at `clipboard_copy.rs:507-512` add `CopyEnvironment` and `tmux_clipboard_copy_ready` to the `mod tests` scope. The PR body claims "coverage for tmux-preferred, fallback, and combined-failure paths," and the diff shows new test cases threading the `CopyEnvironment` struct through `copy_to_clipboard_with` with stub copy fns. I can't see the full test bodies in the diff window I inspected, but the structural setup is correct (deterministic stubs for each of the four backends).

## Risks

- **OSC 52 max-bytes interaction.** The test imports `OSC52_MAX_RAW_BYTES` at `:508` — there's likely an existing size cap on OSC 52 (terminals truncate at ~75KB or similar). The new tmux path has no such cap (`tmux load-buffer` happily takes megabytes), which is actually an improvement for users copying long shell-output, but it means behavior diverges between the tmux and OSC 52 fallback paths in ways the user may not realize. Worth a short comment in the module doc.
- **`tmux info` parsing fragility** (see `Ms: [missing]` substring above). Low blast radius — failure mode is "we send to tmux even when forwarding is broken, user gets buffered text inside tmux but not on system clipboard, which is the *current* broken behavior anyway." Not a regression, just a missed opportunity to report the true problem.
- **No `tmux` binary detection.** If the user has `TMUX` set but no `tmux` binary on PATH (e.g., they're inside a tmux session whose binary was uninstalled, or an environment-variable mock), `Command::new("tmux").spawn()` returns an `Err` immediately and the code falls back to OSC 52 with `"failed to spawn tmux: ..."` in the error string. This is fine — the fallback works — but it's worth confirming via test that the `spawn()` error path doesn't panic.
- **Concurrency with paste.** `clipboard_copy.rs` and `clipboard_paste.rs` both shell out to `tmux`. If a concurrent paste is mid-flight, tmux serializes them itself, so this is fine. Just worth flagging that `STDERR_SUPPRESSION_MUTEX` at `:30` exists for a reason and shouldn't be removed.

## Suggestions

1. Add a `wait_timeout(Duration::from_secs(2))` or equivalent around `wait_with_output()` at `clipboard_copy.rs:354` so a wedged tmux server can't hang `/copy`. Falling through to OSC 52 on timeout matches the existing fallback policy. ([Reference: `wait-timeout` crate is already used elsewhere in codex-rs IIRC; reuse if available.])
2. Pin a tmux version comment near the `Ms: [missing]` parser at `clipboard_copy.rs:393` — something like `// Verified against tmux 3.3a / 3.4 / 3.5; tmux info output format may change.`. If you want to be belt-and-suspenders, also add an alternate parse for `tmux display-message -p '#{client_tty_features}'` which is a more structured query for terminal capabilities.
3. Document the size-cap divergence between OSC 52 (`OSC52_MAX_RAW_BYTES`) and tmux paths in the module doc at `clipboard_copy.rs:5-15`. Users debugging "I copied 200KB and only the first 75KB landed" will have a much easier time if the doc says "OSC 52 has a per-terminal size cap; tmux path does not."
4. Consider exposing the chosen path (tmux/osc52/arboard/wsl) in the success message or as a `tracing::debug!` on the success path — currently every path is silent on success and only `tracing::warn!`s on failure, so users have no way to know which mechanism actually worked when reporting bugs.
5. Add a test case for `set-clipboard external` to lock in that "external" passes the precheck (it should, per the analysis above; explicit test prevents future regression on a stricter `==` check).

## Verdict

**merge-after-nits** — the design is sound, the dispatch ladder is clean, the precheck correctly distinguishes "tmux clipboard works" from "tmux clipboard silently drops," and the test scaffolding is well-shaped. The asks are hardening (subprocess timeout, tmux-version pinning) and observability (which path won), not behavior changes. Once the timeout is added (a real availability concern for a TUI feature) and the tmux-info parser has a version-comment, this is a solid fix for the long-standing #19926.

## What I learned

The right way to handle "feature works in environment A but silently fails in environment B" is rarely to flip a global flag — it's to add a *capability detection* step that resolves the question at runtime. This PR's `tmux_clipboard_copy_ready` is a textbook example: instead of asking the user to set `allow-passthrough on` (config burden) or guessing (silent failure), it asks tmux directly whether forwarding is wired up, and falls through gracefully when it isn't. That pattern — "detect, prefer, fall back, report which path won" — generalizes to any I/O surface where the same logical operation has multiple physical mechanisms (clipboard, notifications, file-watching, terminal queries). The one ingredient this PR is missing for the full pattern is the "report which path won" step, which I suggested as a `tracing::debug!` — that's the difference between a feature that works and a feature that's *debuggable when it doesn't*.
