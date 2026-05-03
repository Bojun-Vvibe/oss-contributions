# block/goose PR #8966 — chore(deps): bump cliclack from 0.3.8 to 0.5.4

- Head SHA: `9d9713d85d00`
- Author: `dependabot[bot]`
- Files changed:
  - `Cargo.lock`
  - `crates/goose-cli/Cargo.toml` (1 line)

## Analysis

Two-version-major dependabot bump for `cliclack` (interactive CLI prompts). Notable consequence visible in the lockfile: `cliclack 0.5.4` now depends on `console = "0.16.x"`, which lets the workspace collapse from two `console` versions down to one.

Diff highlights:
- `crates/goose-cli/Cargo.toml:28` — `cliclack = "0.3.5"` → `"0.5.4"` (note: previous spec was `0.3.5`, lockfile was already at `0.3.8` because of caret resolution).
- `Cargo.lock:1879-1885` — `cliclack 0.3.8` → `0.5.4` with new checksum `4529f45438fc25ca048b242d5c48e2d3ce9a521e2a5a9123d9737d8520b030dd`.
- `Cargo.lock:2053-2065` — entire `console 0.15.11` package entry deleted.
- `Cargo.lock:947, 4479, 5332, 5364` — four other dependents (`bat`, `goose-cli`, `indicatif`, `insta`) lose the disambiguating ` 0.16.3` suffix and now resolve to a single `console` package.

That dedup is the real win: it shrinks compile time, code size, and removes the chance of two `console` ABIs being linked into the same binary (which historically has caused subtle terminal-handling bugs). Worth more than just the version bump itself.

Risks:
- 0.3 → 0.5 is two breaking releases of `cliclack`. Dependabot does not check call-site compatibility; the human reviewer needs to make sure goose-cli's prompt code (search for `cliclack::` in `crates/goose-cli/src/`) still compiles against the new API. The PR body shows only "See full diff" with no changelog excerpt, which is unhelpful — the version notes should be inspected.
- No CHANGELOG / compatibility-score is rendered in the body (Dependabot rendered just the badge), so confidence is purely from `cargo build` + `cargo test` in CI.
- `crates/goose-cli/Cargo.toml` still has `console = "0.16.1"` declared explicitly, which is fine and matches the dedup target.

## Verdict

`merge-after-nits`

## Nits

- Before merging, confirm `cargo check -p goose-cli` is green locally — `cliclack 0.5` removed a few `select`/`multi_select` builder methods in 0.4.0 and renamed an internal trait in 0.5.0.
- Add a one-liner to the PR body summarizing the API breaks the maintainer had to absorb (or "no API changes needed" if the build is clean) so this is greppable in `git log` later.
