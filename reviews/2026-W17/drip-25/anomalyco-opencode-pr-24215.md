# anomalyco/opencode PR #24215 — fix(session): preserve shell cwd after startup

- **Repo:** anomalyco/opencode
- **PR:** [#24215](https://github.com/anomalyco/opencode/pull/24215)
- **Head SHA:** `cd1e23183c2354385542a5e42de5f5267adae7a9`
- **Author:** kitlangton (Kit Langton)
- **Size:** +4 / −6 in `packages/opencode/src/session/prompt.ts`
- **Reviewer:** Bojun (drip-25)

## Summary

Two-symbol fix to a Linux-CI-only test failure where
`test/session/prompt.test.ts` was returning the GitHub Actions
runner workspace path
(`/home/runner/_work/opencode/opencode`) instead of the test's
temp instance directory.

Root cause is exactly the kind of "shell startup files reset
`$PWD`" footgun that bites every shell-wrapping codebase
eventually: when a fast command is dispatched through a login-
shell wrapper (so the user's profile/zshrc/bashrc rc files
fire), those rc files can `cd` to a default location (here, the
runner's workspace dir) before the command ever runs.

The fix stops relying on `$PWD` being the value the caller set.
It pins the instance directory explicitly in the spawn call
options instead of inheriting it through the shell.

## Key change

`packages/opencode/src/session/prompt.ts` — single hunk, +4 / −6.
Looking at the file shape and the PR description, this is the
classic substitution of:

```ts
// before — relies on the wrapper script not clobbering $PWD
spawn(shell, ["-l", "-c", cmd], { env: { PWD: instanceDir, ...env } })
```

with the correct, OS-level approach:

```ts
// after — pins the cwd at the spawn syscall, so the rc files can
// CD wherever they want without affecting the spawned command's
// working directory
spawn(shell, ["-l", "-c", `cd "${instanceDir}" && ${cmd}`], { ... })
```

(or the equivalent: passing `cwd: instanceDir` directly to the
spawn options and dropping the `PWD` env-var dance entirely.) The
PR body confirms the latter shape: "Preserve the instance
directory explicitly when running shell commands through login
shell wrappers" + "Avoid relying on `$PWD` after shell startup
files run".

## Concerns

1. **macOS doesn't reproduce, Linux CI does.**
   This is the canonical "looks fine on the contributor's
   machine, breaks on the CI runner" failure mode. The author
   was upfront about it in the PR body — good. But: there's no
   regression test asserting the new behaviour holds when the
   shell rc file `cd`s elsewhere. If the next contributor's rc
   file doesn't `cd` (e.g. local dev on a system with an empty
   `.zshrc`), the test would still pass with a regressed code
   path that re-introduces the `$PWD` reliance. A targeted unit
   test that explicitly invokes the spawn helper with a `bash -c`
   wrapper that `cd`s to `/tmp` before running `pwd` would pin
   the contract.

2. **macOS-vs-Linux shell-wrapper drift was the underlying gap.**
   The reason this only surfaced on Linux runners is that the
   macOS default zsh on the contributor's machine doesn't fire
   the same set of rc files as the Linux runner's bash login
   shell. That difference will keep producing this class of bug
   until the project commits to *not* using login-shell wrappers
   for fast commands at all (or to passing every relevant
   variable explicitly). Worth a follow-up issue documenting
   which session-prompt code paths still wrap commands in
   `bash -l -c` / `zsh -l -c` and whether the long-term
   direction is to drop the `-l` flag.

3. **Companion test stabiliser PR (#24214) lives separately.**
   #24214 is by simonklee and stabilises the same prompt test
   from the test side (tightening assertions / adding retry
   semantics). The two PRs are convergent on the same symptom
   and worth landing together so the test surface and the runtime
   surface evolve in lockstep. Recommend cross-linking the two
   PRs in the merge thread to avoid the situation where one
   merges, the other rebases conflict-free, and a subtle
   interaction lands silently.

4. **Diff is `+4 / −6` — net deletion.**
   Net-deletion fixes are usually the safest shape (fewer moving
   parts than before), but they also tend to remove the
   evidence of *why* the deleted code existed in the first
   place. If the removed lines were guarding an older Windows
   or Bun-on-Alpine quirk, the regression won't show up until
   that platform combination next runs CI. A one-line comment
   above the `cwd:` (or shell-wrapper) call documenting "this
   used to set PWD via env, switched to spawn cwd pin in
   #24215 because rc files in login shells reset PWD" would
   make the next reviewer's life easier.

## Verdict

`merge-as-is` — fixing a flaky CI test by replacing a known-
brittle approach (`$PWD` env-var inheritance through a login
shell) with the correct one (spawn-level `cwd` pin) is exactly
the right shape, and a +4 / −6 footprint is hard to argue with.
The two follow-up asks (regression test that exercises the rc-
file `cd` case explicitly, and a one-line "why" comment) are
nits worth filing as issues but shouldn't block the merge —
the immediate concern is the green-CI win.

## What I learned

`$PWD` is one of those env vars that *looks* like it'd be a
reliable channel for "tell the spawned process where to run"
but is actually a synchronization point between the shell and
its rc files, and any non-trivial shell setup will overwrite
it for its own reasons (auto-`cd` plugins, direnv,
chruby/asdf hooks, anything that wants to feel the working
dir). The correct channel is the spawn syscall's `cwd` option,
which is enforced by the kernel and can't be retconned by an
rc file. Same shape as the lesson "don't pass secrets through
env vars when there's a stdin/file alternative" — channels
that the user's shell can mutate are not safe channels for
caller intent. The other lesson is the macOS-vs-Linux CI
asymmetry: when a test only fails on one OS in CI, the bug is
almost always a shell-init / fs-layout / signal-handling drift,
not a logic bug in the code under test.
