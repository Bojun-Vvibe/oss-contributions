# openai/codex#21285 — fix(bwrap): emit libcap after standalone archive

- **Head SHA**: `6795cc9f1e6f901706796ba7f9af8ff2a5660b54`
- **Stats**: +7 / -0, 1 file (`codex-rs/bwrap/build.rs`)

## Summary

`build.rs` was using `pkg_config::Config::probe("libcap")` which by default emits `cargo:rustc-link-lib=cap` *immediately* via `cargo_metadata`. But `bwrap` is built into a static archive (`standalone_bwrap.a`) compiled later in the same build script via the `cc` crate; on Linux toolchains that resolve symbols left-to-right (the default), placing `-lcap` before `-lstandalone_bwrap` means the bwrap archive's references to `cap_*` symbols aren't satisfied (the linker has already moved past `libcap` by the time it sees them), causing `undefined reference to cap_get_proc` at the final link.

## Specific citations

- `codex-rs/bwrap/build.rs:38`: adds `.cargo_metadata(false)` to the `pkg_config::Config::new()` chain, suppressing pkg-config's automatic `cargo:rustc-link-lib=` emission so the `-lcap` flag is no longer emitted before the `cc::Build::compile()` call below.
- `:69-74`: after `build.compile("standalone_bwrap")` (line 68) — which emits the `cargo:rustc-link-lib=static=standalone_bwrap` for the bwrap archive — the deferred linkage is now manually re-emitted: `cargo:rustc-link-search=native=<path>` for each `link_paths` entry and `cargo:rustc-link-lib=<name>` for each `libs` entry. This places `-lcap` *after* `-lstandalone_bwrap` in the rustc command line, matching left-to-right symbol resolution requirements.

## Verdict

**merge-as-is**

## Rationale

Textbook fix for a real and well-known link-order bug. The ordering invariant is correct (static archives must precede their dependencies on Unix linkers; `--start-group`/`--end-group` would be the alternative but is uglier and toolchain-specific), and the implementation is the minimal-disruption choice — keeps `pkg_config` as the source of truth for *where* libcap lives and *what it's called*, just defers *when* that information reaches cargo. The two for-loops at `:69-74` correctly mirror what `cargo_metadata: true` would have emitted, just sequenced after `compile()`. No test is needed — this is a build-system fix, and the only meaningful test is "does the bwrap binary still link cleanly?" which the existing CI link step covers. Ship it.
