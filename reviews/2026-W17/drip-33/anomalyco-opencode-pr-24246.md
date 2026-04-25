# Review: anomalyco/opencode#24246 — Preserve nix/direnv PATH in login shell for ! commands

- **PR**: https://github.com/anomalyco/opencode/pull/24246
- **State**: OPEN
- **Author**: xthreehao (Xavier)
- **Range**: +4 / −0
- **Head SHA**: `cdeda3f3a0b2b41ed430c1d0229dfd71bdf47847`
- **Base SHA**: `e29058c346e50976c6b5a2277f22d1902917e65c`
- **Closes**: #6468
- **Verdict**: request-changes
- **Reviewer date**: 2026-04-25

## What the PR does

`packages/opencode/src/session/prompt.ts` runs user `!` commands
through `zsh -c -l` (or `bash -c -l`). The shell sources `~/.zshrc`
/ `~/.bashrc`, which on Nix/direnv setups overrides `PATH` to a
distro-default and drops the dev-environment additions
(`nix develop`, `direnv allow`). The fix saves the inherited
`PATH` into `_opencode_path` before sourcing the rc files, then
re-prepends it after sourcing:

```sh
_opencode_path="$PATH"
[[ -f ~/.zshrc ]] && source ... || true
export PATH="$_opencode_path:$PATH"
```

## Observations

1. **The intent is right; the placement is wrong.** The user's
   Nix/direnv `PATH` does need to win over rc-file overrides.
   Saving and re-prepending is the standard idiom. Good
   instinct.
2. **`-l` (login shell) is the wrong loader for `~/.zshrc`.**
   The current code at `prompt.ts:1664` already invokes
   `zsh -c -l`, then *also* sources `~/.zshenv` and `~/.zshrc`
   manually. With `-l` zsh already runs `~/.zprofile` (and
   `~/.zlogin`), so this PR's diff stacks rc-loading on top of
   login-loading. The resulting precedence is:
   `inherited PATH` → zprofile → zshenv → zshrc → re-prepend
   inherited. If the user's Nix `PATH` was set by `direnv`
   *via* `~/.zshrc` (which is the common direnv hook
   placement), then after this fix the inherited PATH wins
   over the direnv-provided PATH — exactly the wrong direction
   for users who rely on a per-shell direnv hook rather than
   the parent process inheriting it.
3. **The `bash` branch breaks `shopt -s expand_aliases`
   intent.** The bash variant adds `_opencode_path="$PATH"`
   *before* `shopt -s expand_aliases` (`prompt.ts:1677`). That
   ordering is fine, but: `bash -c -l` in non-interactive mode
   doesn't actually source `~/.bashrc` unless `BASH_ENV` is
   set, and `expand_aliases` only matters in interactive
   shells unless explicitly enabled. The existing
   `shopt -s expand_aliases` keeps that working, but this PR
   doesn't change that — it just saves PATH around it.
   Mention here only because the PR's verification claim
   ("Tested in a normal terminal") suggests the author tested
   interactively, not the way `prompt.ts` actually invokes the
   shell.
4. **`export PATH="$_opencode_path:$PATH"` doubles entries.**
   Pre-pending the inherited PATH onto the rc-modified PATH
   means every directory that was in *both* now appears twice
   (e.g. `/usr/bin` will be in `_opencode_path` and re-added
   by `/etc/profile`). This is harmless for resolution
   correctness but inflates `PATH` linearly with each `!`
   invocation if the variable were ever round-tripped. Not a
   bug, but worth using a dedupe pass:
   ```sh
   export PATH="$(printf '%s\n' "$_opencode_path:$PATH" | awk -v RS=: '!seen[$0]++' | paste -sd: -)"
   ```
   Or simpler: just `export PATH="$_opencode_path"` and skip
   the rc PATH entirely if the goal is "inherited wins".
5. **Issue #6468 deserves a config flag.** Two different
   user populations exist: (a) "I want my rc files to set my
   PATH" (current behavior); (b) "I want the parent
   environment's PATH to win" (this PR's behavior). Hard-
   coding (b) silently breaks (a). A
   `shell.preserve_inherited_path: bool` config (default
   matching current behavior, opt-in for Nix/direnv users)
   would ship the fix without flipping the default for users
   who never asked.
6. **No tests.** The change is in shell-script-as-string, so
   it can't be unit tested directly, but a fixture-based test
   that checks `prompt.ts` emits the expected script for each
   shell would catch regression and would have caught the
   PATH-doubling concern.

## Verdict reasoning

Real bug, well-motivated, minimal patch — but it silently
flips the precedence of "rc-set PATH" vs "inherited PATH" for
all users, not just Nix/direnv ones. That's a behavior change
that needs either (a) a config flag, or (b) a more surgical
fix (e.g. only re-prepend specific known-Nix prefixes, or only
when `IN_NIX_SHELL` / `DIRENV_DIR` is set in the inherited
env). Recommend `request-changes` for the precedence
discussion, then it's a one-line add to gate on
`[[ -n "$IN_NIX_SHELL$DIRENV_DIR" ]]`.

## What I learned

`PATH` precedence in shells is one of the few places where
"obviously correct fix" is actually a policy choice in
disguise. Whenever a fix changes which side of a `:`-join
wins, it needs to be either configurable or gated on a
detectable signal — silent precedence flips break someone.
