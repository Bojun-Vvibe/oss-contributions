# google-gemini/gemini-cli#26274 — fix(cli): allow installing extensions from ssh repo

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26274
- **Head SHA**: `235e3d9351e2d889f93fd51be48c6ecdba4ac32e`
- **Size**: +2 / -1, 1 file
- **Verdict**: **merge-as-is**

## Context

`inferInstallMetadata(source, ...)` in `extension-manager.ts` decides whether a `gemini extensions install <source>` invocation treats the source as a git URL. The function had a chain of `startsWith` checks for known git-URL schemes — `https://`, `git://`, `git@`, `sso://`, `github:`, `gitlab:` — but no `ssh://` arm, so users with explicit `ssh://git@host/owner/repo.git` URLs (the canonical SSH-protocol git URL form) were rejected as not-a-git-source.

## Design analysis

### Diff (`packages/cli/src/config/extension-manager.ts:1300-1305`)

```ts
source.startsWith('git@') ||
source.startsWith('sso://') ||
source.startsWith('github:') ||
source.startsWith('gitlab:') ||
source.startsWith('ssh://')   // ← added
```

One-line addition to the existing OR-chain. Purely additive — no existing path changes behavior, only the previously-rejected `ssh://...` source is now accepted as a git source.

## What's right

- **Surgical scope.** One predicate added, no surrounding refactor. Easy to revert if `ssh://` shape needs more nuanced handling later.
- **Symmetric with the `git@` arm.** `git@host:owner/repo.git` (SCP-style) was already accepted — `ssh://git@host/owner/repo.git` (URL-style) is the same protocol, just a different syntactic surface. Both should be accepted.
- **Order of checks doesn't matter** — they're all OR'd, and the predicates are mutually disjoint (different prefixes), so adding `ssh://` at the end is fine.

## Nits

- **No test arm in the diff.** The PR is one line, but a test asserting `inferInstallMetadata('ssh://git@github.com/owner/repo.git', ...)` returns `{ source, ... }` (vs. throwing or returning a non-git shape) would prevent a future predicate-chain rewrite from silently dropping the new arm. The existing test file likely has a parameterized test for the other prefixes — the new one slots in trivially.
- **`sso://` prefix is in the chain** — that's the Google-internal SSO-tunneled-git scheme. Worth confirming the user-facing docs list both `ssh://` and `sso://` so users know which scheme to use when. Doc update could be a follow-up.
- **No validation of the URL beyond `startsWith`.** `ssh://` followed by garbage will pass this check and fail downstream at git clone time. Acceptable — the function is `inferInstallMetadata`, not `validateInstallSource` — but worth a one-line `// shape validated downstream by git clone` comment if the next reader wonders.

## Verdict

**merge-as-is** — minimal, correct, fills a real user-reported gap. The one-line change closes the protocol-shape gap symmetrically with the existing `git@` SCP-style arm. A follow-up test would be welcome but not blocking for a one-line additive predicate.
