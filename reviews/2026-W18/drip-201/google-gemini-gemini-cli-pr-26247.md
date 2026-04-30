# google-gemini/gemini-cli #26247 — fix: expand template vars in MCP stdio config

- **Author:** haosenwang1018 (Sense_wang)
- **SHA:** `00a8540`
- **State:** OPEN
- **Size:** +67 / -4 across 2 files (`packages/core/src/tools/mcp-client.ts`,
  `packages/core/src/tools/mcp-client.test.ts`)
- **Verdict:** `merge-after-nits`

## Summary

Closes a template-expansion gap where `{{HOME}}` / `{{X}}` mustache-style placeholders
in MCP stdio config (`command`, `args`, `cwd`, and `env` values) were left literal
while the same placeholders inside the `env` *values* were already being expanded by
the existing `expandEnvVars(...)` path. New helper `expandMcpServerConfigValue(value,
env)` at `packages/core/src/tools/mcp-client.ts:1168-1178` does the two-pass
expansion: first the mustache `{{NAME}}` regex `/\{\{(\w+)\}\}/g` resolving against
the merged env (with `match` fall-through if the key is missing), then the existing
`expandEnvVars` for `$NAME` / `${NAME}` syntax. Applied uniformly at
`createTransport` `:2293-2304` to `command`, `args[i]`, `cwd`, and `env[*]`. Test at
`mcp-client.test.ts:1996-2032` covers the cross-syntax case (`{{HOME}}/.bun/bin/bun`
in command, `$HOME/config.json` in args, both `{{HOME}}` and the implicit
`expandEnvVars` of remaining `$HOME`).

## Reasoning

The fix is correct in shape: a single helper is applied at all four config-value
sites (command/args/cwd/env), so the expansion semantics are uniform across the
stdio config surface — that was the underlying inconsistency that caused the bug
(only `env` values were being expanded). The `match` fall-through at
`:1173-1175` (`(match, name) => env[name] ?? match`) is the right safety
behavior: an unresolved `{{FOO}}` stays literal rather than collapsing to empty
string, which avoids accidentally turning `{{MISSING}}/bin/cmd` into a path that
resolves to root. Three nits before merge: (1) the regex `\w+` allows underscores
and digits but not dots or hyphens, which is fine for shell-style env-var names but
worth documenting since `{{user.home}}` would silently not match — a `// only
[A-Za-z0-9_] segment names are expanded` comment at `:1170` would prevent the
"why didn't my template work" support thread; (2) precedence is now "mustache first,
then `$VAR`", which means an env value of `${{FOO}}` would expand `{{FOO}}` first
(to its env value, say `BAR`) and then `expandEnvVars` would NOT re-expand `$BAR`
(it's already a literal value, not `$BAR`); a regression locking the precedence
ordering would prevent a future swap from quietly breaking; (3) test only covers
the happy-path expansion — add a negative case for "missing template var stays
literal" since that's the load-bearing safety property. None of these are blocking.
