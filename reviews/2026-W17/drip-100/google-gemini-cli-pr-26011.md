# google-gemini/gemini-cli#26011 — fix(cli): propagate TLS env vars from .gemini/.env in parent process

- **Author**: gaurav0107
- **Size**: +345 / -2
- **Issue**: #25987

## Summary

After PR #24667 split the gemini-cli into a lightweight parent + heavy child process model, `NODE_EXTRA_CA_CERTS` and other TLS-init env vars defined in `.gemini/.env` stopped working. Root cause: Node reads those vars **once at process start**, in C++ before any user JS runs. Pre-#24667, `.env` loading happened before any TLS connection. Post-#24667, the child is spawned with a pre-built env that doesn't yet include the `.env` contents — by the time the child's JS calls `loadEnvironment()`, the TLS layer is already initialized without the cert.

Fix: parent hand-parses `$HOME/.gemini/.env` for a small allowlist and injects matched values into the child spawn env (without overriding the shell). Avoids importing `dotenv` in the parent (would defeat the lightweight-parent goal).

Allowlist (per PR body):
- `NODE_EXTRA_CA_CERTS`
- `NODE_TLS_REJECT_UNAUTHORIZED`
- `SSL_CERT_FILE`, `SSL_CERT_DIR`
- `HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`

## Diff inspection

`packages/cli/index.test.ts` (new, 197 lines) is the substantive review surface — the parser and loader logic is exercised through tests, not visible in the truncated diff. Test coverage:

**`parseSimpleEnv`:**
- `KEY=VALUE` lines ✓
- Strips matching `"..."` and `'...'` quotes ✓
- Ignores blank lines and `#` comments ✓
- Supports `export NODE_EXTRA_CA_CERTS=...` prefix ✓
- Strips inline `# comment` only on **unquoted** values (`BAZ="quoted # stays"` preserves the `#`) ✓
- Drops malformed lines (`=novalue`, `no equals sign`, `9BAD_KEY=nope`) ✓
- Handles CRLF line endings ✓
- Preserves `=` inside values (URL test) ✓

**`loadTlsEnvFromGemini`:**
- Reads `NODE_EXTRA_CA_CERTS` from `$HOME/.gemini/.env` ✓
- Reads multiple allowlisted vars ✓
- **Does NOT override keys already set in `currentEnv`** (shell wins) — the right precedence ✓
- Skips non-allowlisted keys (`FOO`, `GEMINI_API_KEY`) — critical for the security model ✓
- Returns `{}` when no home `.env` exists ✓
- **Does NOT read project-local `.gemini/.env`** with explicit comment about the trust model — child handles project-scope under workspace-trust ✓
- Tolerates malformed `.env` content without throwing ✓

The "trust-gated, child handles it" comment at `index.test.ts:160-168` is the most important architectural justification — the parent can't cheaply verify workspace trust, so it deliberately stays HOME-only. This prevents a malicious project's `.gemini/.env` from injecting `NODE_EXTRA_CA_CERTS` to defeat cert validation.

## Strengths

- Hand-parser (no `dotenv` import) preserves the lightweight-parent contract.
- Allowlist approach (vs blanket pass-through) limits blast radius — only the seven vars Node/libcurl actually need at init time.
- Shell-wins precedence (`currentEnv` not overridden) matches user intuition: `NODE_EXTRA_CA_CERTS=foo gemini` should beat `.env`.
- HOME-only (not project-local) is the right trust boundary at the parent layer — well-documented in the test comment.
- Test coverage is thorough: edge cases (CRLF, `=` in values, escaped `#` in quoted strings, malformed lines, missing file, empty file).
- The `processSimpleEnv` parser correctly handles the `export` prefix — common gotcha for hand-rolled `.env` parsers.

## Concerns

1. **The actual production code is not visible in the truncated diff** — `parseSimpleEnv`, `loadTlsEnvFromGemini`, and the spawn-env injection logic must be in `packages/cli/index.ts`. Need to verify:
   - The injection happens **before** `child_process.spawn(..., { env })` is called.
   - The injection doesn't accidentally overwrite child-process `PATH` or other critical vars.
   - `PARENT_PROCESS_TLS_ENV_ALLOWLIST` is exported from `index.ts` (the test imports it).
2. **No test for the `NO_PROXY` precedence interaction.** `HTTP_PROXY=...` + `NO_PROXY=corp.example.com` is a common corp setup; the test should pin that both vars survive injection together.
3. **`NODE_TLS_REJECT_UNAUTHORIZED=0` is in the allowlist.** This is a footgun — a project (or any malicious actor with HOME write access) can disable cert validation globally for the spawned child. Consider a `console.warn(...)` when this specific var is injected from `.env` (not from shell), so the user sees "your `.env` is disabling TLS validation" once per session.
4. **Project-local `.gemini/.env` skip is correct, but the user impact isn't documented.** A user who puts `NODE_EXTRA_CA_CERTS=./certs/ca.pem` in their **project** `.env` will see "it works in single-process mode but stops working after upgrade" — the migration guide / release notes should call this out explicitly.
5. **Symlinked `~/.gemini/.env`** behavior isn't tested. `fs.realpathSync(fs.mkdtempSync(...))` is used to make the test stable on macOS, but the production code path with a user-symlinked `.env` (common for dotfiles managers) needs a comment confirming `fs.readFileSync` follows symlinks (it does, but worth a one-line note).
6. **No size cap on `.env` parsing.** A pathological 100MB `.env` would be read into memory in the parent (bypassing the lightweight-parent goal). A `fs.statSync(path).size > 64*1024 → bail` guard is cheap insurance.

## Verdict

**merge-after-nits** — the architecture is sound, the parser is well-tested, and the trust-model comment is the right call. Need to (a) eyeball the production code in `index.ts` for spawn-env merge correctness, (b) add the `NODE_TLS_REJECT_UNAUTHORIZED` warn-on-inject, (c) document the project-local `.env` migration footgun, and (d) add a `.env` size cap. None are merge-blocking individually but together justify a follow-up commit before landing.
