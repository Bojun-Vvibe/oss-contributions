# openai/codex #19931 — Move local resume cwd filtering into thread/list

**Verdict:** `merge-as-is`

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/19931
- Head SHA: `4721d365`
- Diff: +55 / -20, single file `codex-rs/tui/src/resume_picker.rs`

## What it does

Pushes local-mode `cwd` filtering for the `/resume` and `/fork` pickers
*down* from the TUI (which was scanning loaded rows after the fact) into
the `thread/list` app-server call (which can prune at the source). Affects
the no-arg paths: `/resume` from inside the TUI, `codex resume`, `codex
resume --all`, `codex fork`, `codex fork --all`. The single-id paths
(`codex resume <id>`, `codex resume --last`, etc.) are explicitly out of
scope — they don't go through this picker.

## Design analysis

Right shape and the change is well-factored. The new `picker_cwd_filter()`
helper at `tui/src/resume_picker.rs:271-284` consolidates the
three-cell decision matrix (`show_all`, `is_remote`, `local`) into one
place, and both call sites — `run_resume_picker_with_app_server` at `:157`
and `run_fork_picker_with_app_server` at `:182` — now route through it
instead of inlining different versions of the same conditional:

```rust
fn picker_cwd_filter(
    config_cwd: &Path,
    show_all: bool,
    is_remote: bool,
    remote_cwd_override: Option<&Path>,
) -> Option<PathBuf> {
    if show_all {
        None
    } else if is_remote {
        remote_cwd_override.map(Path::to_path_buf)
    } else {
        Some(config_cwd.to_path_buf())
    }
}
```

The previous two call-site copies were textually identical
(`if show_all { None } else { app_server.remote_cwd_override().map(...) }`)
but they silently lost the local case — they only filtered when the user
had passed `--cd` to a remote, never when running locally. Local cwd
filtering was instead happening *after* pages came back, in
`run_session_picker_with_loader` at the deleted `:218-225` block:

```rust
-    let filter_cwd = if show_all || is_remote {
-        None
-    } else {
-        std::env::current_dir().ok()
-    };
+    let filter_cwd = None;
```

Pushing the filter into `thread/list` is the right level: the server has
the index, can return the right page count immediately, and avoids the
"load 50, drop 47, repeat" pagination wastage that the `/// sessions
appear during pagination` doc comment at `:143` was confessing to. The
doc-comment update on the same lines is an honest statement of the new
contract:

```
-/// 1. Provider and source filtering at the backend.
-/// 2. Working-directory filtering at the picker (unless `--all` is passed).
+/// 1. Provider, source, and eligible working-directory filtering at the backend.
+/// 2. Typed search filtering over loaded rows in the picker.
```

Subtle correctness improvement worth calling out: the previous code used
`std::env::current_dir().ok()` for the local filter, the new code uses
`config.cwd.as_path()`. These can differ if the process cwd has changed
since startup (e.g. because a tool ran `cd`), and `config.cwd` is the
intentional, captured-at-startup project root. Using `config.cwd` is the
correct invariant — `/resume` should be filtering by *the project the user
launched in*, not whatever transient cwd the process happens to have.

The new test at `:1575-1592`
(`local_picker_thread_list_params_include_cwd_filter`) pins the load-bearing
cell:

```rust
assert_eq!(
    params.cwd,
    Some(ThreadListCwdFilter::One(String::from("/tmp/project")))
);
```

That's exactly the right assertion — it verifies the `picker_cwd_filter`
output flows through `thread_list_params` into the protocol-level filter
field, and a future regression that drops the local filter would fail
this test.

## Risks

1. **Behavior change for users with local `cwd` symlinks.**
   `std::env::current_dir()` (old) resolves through the OS, while
   `config.cwd` is whatever was captured at process start. If a user
   launched in a symlinked path and the server normalizes paths
   differently, post-PR they may see fewer rows. Probably the right new
   behavior, but worth a release-note line.
2. **Test only covers the `is_remote=false, show_all=false` cell.** The
   helper has three branches; the new test pins one. The other two cells
   (`show_all=true`, `is_remote=true with override`) are pre-existing
   behavior preserved by inspection, but a parameterized test of all three
   cells would prevent the next refactor from silently regressing them.
3. **No backward-compat concern for old session indexes.** Implicitly
   relies on `thread/list` already supporting `ThreadListCwdFilter::One`,
   which it must (the helper exists at the call site already). Not a risk
   in this PR; just noting that this PR does not introduce protocol
   surface, only consumes existing surface.

## Suggestions

- Add the two missing cells to `local_picker_thread_list_params_include_cwd_filter`,
  ideally as a parameterized table test, asserting `params.cwd ==
  None` for `show_all=true` and the remote-override case for
  `is_remote=true`.
- Consider a benchmark or perf-note in the PR body — the author claims
  "slightly faster to load" without numbers, and a "before/after thread
  count vs. time-to-full-page" measurement would justify the move.
- The `pub` boundary doesn't widen here, which is good. No follow-up
  needed on visibility.

## What I learned

This is the textbook "filter at the source, not the sink" cleanup. The
old code was paginating server pages client-side, which is the
load-bearing reason the docstring confessed `sessions appear during
pagination`. Once the source can do the filter, the sink should stop —
and the cleanup of the right-shaped helper plus a unit test pinning the
contract is the right amount of refactor to ship with the move.
