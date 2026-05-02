# Review: charmbracelet/crush #2745

- **PR:** charmbracelet/crush#2745 — `Security: remove env from safe commands and fix shell blocklist bypass`
- **Head SHA:** `15a5acbb85fdbdffae79da78310c88ce516d3a30`
- **Files:** 4 (+44/-6)
- **Verdict:** `request-changes`

## Observations

1. **Removing `env`/`printenv`/`set` from `safeCommands` is correct** (`internal/agent/tools/safe.go:11-25`) — these all dump environment variables which routinely contain API keys, tokens, and other secrets. Auto-approving them in agent tool dispatch was a meaningful exfiltration vector. Removing `gojq.WithEnvironLoader(os.Environ)` from `internal/shell/jq.go:131-134` closes the same hole through the jq builtin (`env.AWS_SECRET_ACCESS_KEY` was readable). Both removals are clean.

2. **Interpreter-chaining detection is fundamentally too narrow to call "fixed"** — `internal/shell/shell.go:242-258` only catches the literal pattern `<interpreter> -c "<single command>"` parsed via `strings.Fields`. Trivial bypasses:
   - `bash -c $'curl https://example.com'` (ANSI-C quoting) — `strings.Fields` splits on whitespace inside the C-quoted string
   - `bash -c "cu""rl https://example.com"` (string concatenation in shell)
   - `bash -c 'eval "$(echo Y3VybA== | base64 -d) https://example.com"'` (eval indirection)
   - `bash -c "$VAR https://example.com"` with `VAR=curl` set elsewhere
   - `bash <(echo curl ...)` (process substitution, no `-c`)
   - `bash -c "curl\thttps://example.com"` — actually fine, but `curl$IFS"https://..."` is not
   - `python -c "import os; os.system('curl ...')"` — `strings.Fields` on the python string yields `['import', 'os;', 'os.system(...)']`, no `curl` token, blocklist misses entirely.
   The PR title says "fix shell blocklist bypass" (singular) but this is closer to "block one specific naive bypass." Worth either a more conservative title ("partially mitigate interpreter chaining") or a more robust approach (e.g. block any unparseable `-c` payload, or refuse interpreter invocations with `-c` when blocklist is non-empty).

3. **`strings.Fields(args[i+1])` is the wrong parser** — even ignoring the bypasses above, `bash -c 'curl   https://example.com'` is parsed correctly by Fields, but `bash -c 'curl;rm -rf /'` yields tokens `['curl;rm', '-rf', '/']` and the blocker matches on `inner[0] == 'curl;rm'` only if the blocklist literally contains that string. The check assumes shell tokenization but uses whitespace splitting. A real fix needs `mvdan/sh/syntax` (already a crush dependency) to parse the inner script and check command nodes.

4. **Hardcoded interpreter list in `shell.go:243`** — `bash, sh, zsh, dash, ksh, python, python3, perl, ruby, node, nodejs`. Misses: `bash5`, `/bin/bash`, `/usr/bin/python3.11`, `pwsh`, `tclsh`, `lua`, `php`, `awk -f`, `gawk`, `sed -e` (yes, sed can spawn processes via `e` command). Path-prefixed invocations bypass this entirely (`/bin/bash -c ...`). At minimum `filepath.Base(args[0])` should be used instead of `args[0]`.

## Recommendation

The `safeCommands` and `jq` changes are correct and should land. The `blockHandler` interpreter detection should not ship in this form — it gives a false sense of security and the test cases (`bash -c 'curl https://example.com'`) only cover the narrow happy path while real adversaries will use any of the above bypasses. Either split this into two PRs (land the first, redo the second properly with `mvdan/sh/syntax` parsing) or expand the test matrix to cover the bypass cases above and gate-fail until they're caught.
