# openai/codex #20654 — fix(tui): bound startup terminal probes

- URL: https://github.com/openai/codex/pull/20654
- Head SHA: `9d2f3b5e149ce075f543ae3bc5a2c7dafea47c5c`
- Files: `codex-rs/tui/src/{custom_terminal,lib,terminal_palette,terminal_probe,tui}.rs` (+503/-21)

## Context / problem

Crossterm's response-probe APIs (`get_cursor_position`, `query_keyboard_enhancement_flags`, `query_foreground_color` / `query_background_color`) are blocking with no caller-controlled deadline. On terminals that don't reply (a long tail: dumb pipes, broken serial consoles, some screen/tmux configs), each unanswered probe stalls TUI startup ~2s; the OSC 10/11 default-color path issues two probes back-to-back, so worst-case startup wedges multiple seconds before any frame paints. Author confirms the manual VM repro: `~2s` per unanswered crossterm probe vs. `~100ms` per bounded `/dev/tty` probe after this fix.

## Design analysis

The PR introduces a private `terminal_probe` module (`terminal_probe.rs:415` net add) that opens `/dev/tty` directly with `OpenOptions::new().read(true).write(true).open("/dev/tty")` (line 158), drives it with raw `libc::poll` against a `pollfd { events: POLLIN, ... }` (line 207-215) under a single per-probe `Duration` budget, and parses the responses inline rather than going through crossterm's blocking event loop. Three call-site swaps:

1. **Initial cursor position** — `custom_terminal::Terminal::with_options_and_cursor_position(backend, cursor_pos)` is added at `custom_terminal.rs:213` so the caller (startup path in `tui.rs`) computes the cursor via the bounded probe *first*, then constructs the terminal with that pre-resolved position. The internal common-path constructor `with_screen_size_and_cursor_position` is factored out so the two entry points share it. Right shape — the caller owns the deadline policy.
2. **Keyboard enhancement detection** — bounded probe with the same poll-based loop.
3. **OSC 10/11 default-color** — at `terminal_palette.rs:80-92`, `cache.get_or_init_with(query_default_colors)` (was: `... || query_default_colors().unwrap_or_default()`) and `cache.refresh_with(query_default_colors)`. `query_default_colors` now lives in `terminal_probe` and uses **one shared deadline for both fg and bg slots** (`terminal_probe.rs:249-253`: `let deadline = Instant::now() + timeout; let Some(fg) = query_color_slot(&mut tty, /*slot*/ 10, remaining(deadline))?; let Some(bg) = query_color_slot(&mut tty, /*slot*/ 11, remaining(deadline))?;`). This is the load-bearing correctness fix — the old code allowed two independent full-budget waits, doubling worst-case stall.

The cache contract (`cache.attempted && cache.value.is_none()` short-circuit at `:88`) is preserved so a one-time failed probe doesn't get re-attempted on every refresh — important since the failure mode is "terminal will *never* answer," not "transient." 

## Risks

- **`/dev/tty` open vs. `stdin/stdout` fds** — opening `/dev/tty` directly is the right choice when stdio is redirected (so probes don't go to a pipe), but on some sandboxed exec environments `/dev/tty` is missing or permission-denied. The `?` in `OpenOptions::open("/dev/tty")?` will bubble up as `io::Error`; need to confirm the call sites treat this as "fall back to default cursor at origin" rather than as fatal startup failure. Looking at the call site, `with_options_and_cursor_position` does take a pre-resolved `Position` so the caller can default to `Position { x: 0, y: 0 }` on error — should be wrapped in a `query_initial_cursor_position(...).unwrap_or_default()` or an explicit `tracing::warn!` on the error path; PR body says "Preserve default-color cache behavior so a failed attempted query does not retry forever" which suggests this is handled, but worth a one-line confirmation in PR body that the cursor probe failure path is also non-fatal.
- **Non-Unix path** — PR explicitly preserves crossterm fallback on non-Unix (Windows). That's the right scope but means the originally-reported "stall on broken terminals" is fixed only on Unix. Worth noting in PR description for downstream packagers shipping a Windows binary.
- **`#[cfg(all(unix, not(test)))]` gating on `terminal_palette::imp`** — the `crossterm::style::query_*_color` imports are now removed from the imp module (`terminal_palette.rs:73-75` deletion), and the bounded probe lives in `terminal_probe`. Worth checking the `not(test)` cfg — does the test path stub `default_colors()` separately, or does the new bounded probe become test-reachable? If the latter, `/dev/tty` open in CI test runners would fail (they don't have a controlling terminal).
- **`assert!`-driven test coverage** — author ran `cargo test -p codex-tui terminal_probe` clean. Good. The pre-existing local stack overflow in `app::tests::discard_side_thread_keeps_local_state_when_server_close_fails` is honestly disclosed as not introduced by this change, which is the right disclosure.

## Suggestions

- Add an explicit `tracing::debug!("terminal probe budget exceeded for {probe_name}, falling back to default")` on the timeout path so a user reporting "weird startup" can `RUST_LOG=codex_tui::terminal_probe=debug` and see exactly which probe gave up.
- Consider a `terminal_probe::PROBE_TIMEOUT: Duration = Duration::from_millis(100);` named constant rather than per-call `Duration::from_millis(100)` literals — easier to tune and to grep.

## Verdict

`merge-after-nits` — correct fix for a real "TUI takes 2s+ to start on broken terminals" symptom, with the right architecture (bounded `/dev/tty` probes, single shared deadline for the dual fg/bg color queries, caller-owned cursor pre-resolution, cache-attempted short-circuit preserved so we don't retry forever). The `+503/-21` diff is justified by a from-scratch probe module that replaces an external dep's blocking primitives. Wants confirmation of `/dev/tty` open-failure fallback and cfg-test reachability.
