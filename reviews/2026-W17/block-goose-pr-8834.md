# block/goose #8834 — Fix Windows dev loop: beforeDevCommand script + Vite IPv4 bind + test_acp_client.py stdin-flush

- **Repo**: block/goose
- **PR**: #8834
- **Author**: seydousakho-star
- **Head SHA**: 7d26c8d048c38afff05299c040c19d0b3fbd7c47
- **Base**: main
- **Size**: +76 / −6 across two files: `test_acp_client.py` (+62/−8)
  and `ui/goose2/justfile` (+18/−2 in two recipes).

## What it changes

Three Windows-specific fixes to the inner-loop developer experience,
each documented inline with the *why*:

1. **`test_acp_client.py:11-22`** — UTF-8 console reconfig at startup
   so progress glyphs (`✓ ✗ 📝`) don't crash on cp1252 Windows
   consoles. Wrapped in try/except for older Pythons.

2. **`test_acp_client.py:25-49, 75-93`** — `_resolve_goose_cmd()`
   helper picks `$GOOSE_BIN` → `target/debug/goose[.exe]` →
   `cargo run -p goose-cli -- acp` in that order. Subprocess
   construction switches `stderr=subprocess.PIPE` →
   `subprocess.DEVNULL` and `bufsize=0` → `bufsize=1`, with comments
   at `:73-83` explaining that an unread `cargo run` stderr fills
   Windows' ~4KB pipe buffer and deadlocks the JSON-RPC stdio loop.

3. **`ui/goose2/justfile:99-115` and `:147-166`** — Tauri
   `beforeDevCommand` config rewritten:
   - Drops `cd ${PROJECT_DIR} && ` prefix (the embedded `&&`
     was getting parsed by the bash→npm→tauri argv passthrough
     and truncating the JSON value).
   - Drops the leading `exec` (cmd.exe doesn't know the bash
     builtin).
   - Adds `--host 127.0.0.1` (Vite 7 binds IPv6-only on Windows;
     WebView2 tries IPv4 first and 404s).
   - Changes `cwd` from `.` to `..` (Tauri resolves
     `beforeDevCommand` cwd relative to `src-tauri/`, not where
     `just` ran).

## Strengths

- **Each fix has a comment explaining the failure mode it addresses.**
  This is unusually high-quality for an env-fix PR — the comment at
  `justfile:99-114` enumerates all four sub-fixes and *why* each one
  matters, so a future maintainer untangling another shell-quoting
  issue won't have to re-derive the chain. Same applies to the
  stderr/bufsize comment block at `test_acp_client.py:73-83`.
- The `_resolve_goose_cmd()` precedence (`:25-49`) — explicit env
  override, then prebuilt binary, then `cargo run` — is the right
  ordering for a test that ships in-tree but is sometimes run by
  CI and sometimes by an interactive developer. The `RuntimeError`
  with actionable advice at `:48` (build with `cargo build -p
  goose-cli` or set `GOOSE_BIN`) is good UX.
- `stderr=subprocess.DEVNULL` is the right call for this test:
  the test cares only about JSON-RPC framing on stdout, and
  routing stderr to DEVNULL avoids the pipe-buffer deadlock without
  needing a reader thread. The alternative — spinning up a
  background reader — would add complexity for no test value.
- `bufsize=1` (line-buffered) at `:91` is correct for text-mode I/O
  on both Windows and POSIX. The previous `bufsize=0` is only
  meaningful for binary streams and was actively wrong here.
- `--host 127.0.0.1` is a *less* permissive bind than the default
  (which on Linux is `localhost` resolving to both v4/v6, and on
  Windows happens to be IPv6-only) — explicit IPv4 binding is both
  the cross-platform fix and a small security improvement (no
  accidental external exposure if Vite ever changes the default).

## Concerns / asks

- **The `dev` and `dev-debug` recipes have copy-pasted explanatory
  comments** but only the `dev` recipe lists all four sub-fixes;
  the `dev-debug` block at `:160-163` only covers the `exec` drop
  and `--host` add, not the `cwd` change. Either:
  1. Verify that `dev-debug` doesn't suffer from the `&&` and
     `cwd` issues (and add a one-liner explaining why), or
  2. If both recipes have all four issues, list all four in
     both comment blocks. Otherwise a future maintainer reading
     just `dev-debug` will wonder if the fix is incomplete there.
- The `--host 127.0.0.1` bind change is a *behavior* change for
  *all* platforms, not just Windows. On Linux, devs who were
  reaching the dev server from another box on the LAN (e.g. for
  cross-device testing) will now get connection refused. Worth a
  one-liner in the comment block calling that out, or making the
  host configurable via env var (e.g. `${GOOSE_DEV_HOST:-127.0.0.1}`)
  so the cross-platform default is safe but the LAN-test workflow
  still has an escape hatch.
- `sys.stdout.reconfigure(...)` at `test_acp_client.py:18` swallows
  `AttributeError, OSError`. The `AttributeError` covers
  `reconfigure` not existing on Python ≤ 3.6; the `OSError` covers
  detached/closed streams. Worth a one-line warning print to
  `sys.stderr` (which is now DEVNULL'd inside the subprocess but
  still works at the top of the test script) so a Windows dev who
  *does* hit the cp1252 fallback knows why their output looks
  garbled.
- `_resolve_goose_cmd()` checks `os.path.isfile(prebuilt)` at
  `:43`, but doesn't check that the file is *executable*. On a
  fresh checkout where `target/debug/goose.exe` exists but lacks
  `+x` (rare on Windows, possible on POSIX after a `git restore`),
  the test will fall through to a confusing `Popen` error.
  `os.access(prebuilt, os.X_OK)` would catch it cleanly.

## Verdict

**merge-after-nits** — the actual fixes are correct, well-commented,
and address real Windows-only failure modes that block the entire
inner-loop dev experience there. The asks are: parity between the
`dev` and `dev-debug` comment blocks, an env-var escape hatch for
the IPv4 bind, and a couple of minor robustness tweaks in the
Python helper.

## What I learned

The `cargo run` stderr-deadlock pattern is one of the canonical
subprocess pitfalls and it bites *exactly* the test scripts that
spawn cargo because cargo prints compile chatter to stderr by
default. The fix is always the same — read it on a thread, or
DEVNULL it — but it's worth remembering that pipe-buffer sizes
differ across platforms (Windows ~4KB, Linux ~64KB), so a test
that works on Linux can deadlock on Windows just by being slow
to drain stderr.

The Tauri `beforeDevCommand` quoting issue is a different category
of bug: it's a contract leak between four layers (just → bash →
npm → tauri → cmd.exe). Embedding `&&` in a JSON value passed
across all four layers is a recipe for argv splitting at the
wrong point, and the symptom (truncated `--config` JSON) is
nowhere near the cause (`&&` parsed too early). Per-platform
escape rules accumulate and the only durable fix is to not put
shell metacharacters in cross-process argv values at all.
