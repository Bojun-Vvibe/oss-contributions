# PR #26063 — fix(security): restrict permissions on project temp dir tree

- **Repo**: google-gemini/gemini-cli
- **PR**: #26063
- **Head SHA**: `5001f493`
- **Author**: pmenic (Paolo Menichetti)
- **Size**: +89 / -35 across 12 production files + 1 test file
- **Verdict**: **merge-after-nits**

## Summary

Closes #24743 by tightening permissions on every directory and
file the CLI writes under `~/.gemini/` to `mode: 0o700` for
directories and `mode: 0o600` for files. On shared multi-user
Linux systems this prevents conversation history, agent activity
logs, tool execution output, checkpoints, compression state,
memory state, the project registry, and shadow git checkpoint
repositories from being world-readable. The reporter pinpointed
`ChatRecordingService` only, but the author did a wider audit
and found the same default-umask oversight across 12 files,
including a critical ordering gap: `ProjectRegistry` creates the
per-project slug dir under both `~/.gemini/tmp/<slug>/` and
`~/.gemini/history/<slug>/` *before* the previously-patched
`Storage.ensureProjectTempDirExists()` runs, and because
`mkdir({recursive: true, mode})` is a no-op when the dir already
exists, the later patches were silently ineffective for those
parents — only the leaf `.jsonl` files got `0o600`. The PR
patches the whole tree so the parent dirs land at `0o700` on
first creation regardless of which writer wins.

## Specific changes

- `packages/cli/src/utils/activityLogger.ts:693,706` — `setupFileLogging` now creates the logs dir with `{recursive: true, mode: 0o700}` and appends to the log file with `{mode: 0o600}`. The `appendFile` mode-on-create is the right shape because the file may not yet exist on the first write.
- `packages/core/src/config/projectRegistry.ts:93,101,160,367,387` — five touch-points: the two `mkdir(dir, ...)` calls in `save()` (one in the existence-check branch at `:93`, one unconditional at `:101`), the `mkdir(dir, ...)` in `getShortId()` at `:160`, the per-project `mkdir(slugDir, ...)` at `:367` (which is the one the description names as the race-winning writer), and the `writeFile(markerPath, ..., {encoding, flag: 'wx', mode: 0o600})` at `:387` for the `.project_root` marker. The `writeFile` change preserves the existing `flag: 'wx'` (write-and-fail-if-exists, which is the right idempotency for a marker file) and just adds `mode` alongside.
- `packages/core/src/config/storage.ts:207` — `ensureProjectTempDirExists` adds `mode: 0o700` to the existing `mkdirSync({recursive: true})`. Note this is now redundant when `ProjectRegistry` got there first (which it does) — but it's the right defensive fallback for code paths that bypass the registry.
- `packages/core/src/context/contextCompressionService.ts:96-100`, `packages/core/src/context/processors/toolMaskingProcessor.ts:117,124`, `packages/core/src/context/toolOutputMaskingService.ts:174,192` — same pattern, with the wrinkle that `writeFile` calls that previously passed `'utf-8'` as the third positional argument are migrated to an options object so `mode` can sit alongside `encoding`. That's a syntactic change (no behavior delta if the fs API is sane) but worth noting because a careless string-only diff reader could miss that `'utf-8'` got moved into `{encoding: 'utf-8', mode: 0o600}`.
- `packages/core/src/services/chatRecordingService.test.ts:~1201` — updates the existing `mkdirSync` argument assertion to include `mode: 0o700` and adds a new `describe('file permissions (security)', ...)` block that iterates over every `mkdirSync` and `appendFileSync` mock call and asserts each one carries the security mode. This is the right *intent-pinning* test shape, modeled on the OAuth token storage tests at `oauth-token-storage.test.ts:158-166`. A future refactor that drops `mode:` from any of the patched call sites fails this test loudly.

## Risks

- **`mode:` only applies on creation**: the PR description correctly calls this out — pre-existing dirs and files retain their old (umask-default, typically `0o755`/`0o644`) permissions. Users who installed the CLI before this lands keep the world-readable history until they start a new session in a new project. A retroactive `chmod` on every CLI startup is intentionally out of scope (it has its own EPERM/ownership edge cases on shared mounts), but a release-notes line that tells users "for the security fix to apply to existing data, run `chmod -R go-rwx ~/.gemini/`" would close the surprise gap.
- **Windows is a no-op**: NTFS uses ACLs, not POSIX mode bits, so on Windows the new permissions are inert. The PR description says this matches the existing posture (`FileKeychain`, `oauth-token-storage` are similarly POSIX-only) which is fair, but a Windows multi-user environment running gemini-cli is just as exposed after this PR as before. A follow-up that uses `icacls` or `fs.chmod` with NTFS ACL semantics would be a meaningful second commit; not a blocker for this one.
- **Shadow git internals**: `simple-git` invokes the `git` binary, which creates `.git/HEAD`, `refs/`, etc. with the user's umask. The PR correctly relies on the parent `<repoDir>` being `0o700` so `alice` can't traverse — but if that parent gets `chmod g+x`'d for any reason (a tool that "fixes" permissions, a backup restore, etc.), the inner files become readable again. Worth mentioning in the release notes that the protection is structural, not file-level, for the git tree.
- **Test only covers `ChatRecordingService`**: the PR description argues the other 8 modules use real temp dirs in tests rather than mocking `fs.*`, so adding `mode:` assertions there would require restructuring those tests. That's defensible, but the asymmetric coverage means a future contributor who only runs the touched-test suite for one of those other modules wouldn't catch a mode-strip regression. A single shared `expect(mkdirSync).toHaveBeenCalledWith(expect.anything(), expect.objectContaining({mode: 0o700}))` helper imported into each module's test would amortize the cost.
- **Two `mkdir` calls in `save()` (`:93` and `:101`)**: the second one is described as "Unconditionally ensure the directory exists; recursive ignores EEXIST" — which is true, but having two `mkdir` calls in the same function with a randomized tmp path between them is a code smell that hides the real intent (the first call gates the existence check, the second is defensive against a TOCTOU race). The mode change is correct in both, but a future reader will wonder which one is authoritative; a one-line comment that the dual-mkdir is intentional race-protection would help.

## Verdict

`merge-after-nits` — substantive security fix with a correct,
end-to-end-verified diagnosis (the Docker-based `stat -c "%a %n"`
verification in the description is exactly the right shape for
this class of bug, and the demonstrated `alice → Permission
denied` after the patch vs. full-history dump before is the
strongest possible signal). Asks: (1) add a release-notes line
about the on-creation-only semantic and the chmod recipe for
existing installs; (2) extract the `expect.objectContaining({mode})`
assertion into a shared test helper so future fs.* writes under
`~/.gemini/` get caught even in modules that don't currently
mock fs; (3) file a follow-up issue tracking the Windows ACL
gap so the asymmetry is visible in the issue tracker.

## What I learned

This is a near-textbook case of "the reporter pointed at the
visible symptom, but the bug is structural". The right
investigative move was the Docker-based end-to-end verification
that walks the resulting tree and lists every entry's mode —
that's what surfaced the `ProjectRegistry` race and the
`mkdir(...recursive: true, mode...)` no-op-on-existing trap,
neither of which would have shown up by reading the original
issue or by patching only `ChatRecordingService`. The mode-on-
existing trap in particular is a footgun every Node service that
writes to user-controlled directories has to learn the hard way:
`mode` only applies on creation, and "creation" is decided by
whichever writer wins the race, not by the writer that thinks
it's responsible for the directory.
