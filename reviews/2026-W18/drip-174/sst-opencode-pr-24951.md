# sst/opencode #24951 — enable file watcher in web/serve mode

- **PR:** https://github.com/sst/opencode/pull/24951
- **Title:** `fix(web): enable file watcher in web/serve mode`
- **Author:** tevenfeng
- **Head SHA:** da23977d6e47b139f8385967b4c3ad2f5a0c07c6
- **Files changed:** 2 (`packages/opencode/src/cli/cmd/serve.ts`, `web.ts`), +2 / −0
- **Verdict:** `merge-after-nits`
- **Closes:** #19182

## What it does

Sets `process.env.OPENCODE_EXPERIMENTAL_FILEWATCHER = "true"` at the top of
both the `web` and `serve` command handlers, mirroring what the desktop
Electron entry point already does (`desktop-electron/src/main/server.ts:68`,
per the PR body). Without the flag, `FileWatcher` only subscribes to
`.git/HEAD` for branch detection and never watches the working tree, so
`file.watcher.updated` SSE events never fire and the web "Changes" tab
stays stale until a manual refresh.

## Why this is correct

The desktop path already proved the flag-on behavior works in production;
this PR is just removing a parity gap that broke `web` and `serve` callers
who reasonably expected file watching to work. The placement (very first
statement in `handler`, before the password warning at
`serve.ts:9-11` / `web.ts:34-36`) ensures the env var is set before any
sub-module that reads it lazily.

## What's good

- Two-line patch, identical pattern in both files, no new abstraction.
- Targets the actual handler, not `Bun.spawn` env, so works whether the
  command is launched directly or via `tsx`/`bun run`.
- PR body explicitly references the desktop-mode line as the prior art.

## Nits / risks

- **Mutating `process.env` for a flag named `EXPERIMENTAL_*` is a code
  smell.** The flag is still gated as experimental at the
  `Flag.OPENCODE_EXPERIMENTAL_FILEWATCHER` read site, but the only callers
  that set it are now (a) the desktop main process and (b) these two CLI
  handlers. At that point the "experimental" prefix is misleading — either
  rename the flag (`OPENCODE_FILEWATCHER` with a default-on policy and an
  opt-out) or document why three of three first-party entry points
  unconditionally turn it on.
- **Flip the default instead.** The cleaner long-term fix is to invert
  `Flag.OPENCODE_EXPERIMENTAL_FILEWATCHER` to default-on and let the few
  remaining contexts that don't want it (CI smoke, headless test
  harnesses?) opt out. The three-call-site env-var dance is going to
  rot the next time someone adds a fourth entry point.
- **No test.** The fix is one line in each of two handlers, but a
  `web.test.ts`-style assertion that `process.env.OPENCODE_EXPERIMENTAL_FILEWATCHER === "true"`
  after `WebCommand.handler` runs would catch a future contributor who
  re-orders the handler body and accidentally moves the env-set after the
  module that reads it.
- **Verification claim is anecdotal.** "Started the opencode web server …
  modified a file externally" is fine for a single-developer smoke; the
  PR body doesn't mention whether `serve` mode was also smoke-tested
  end-to-end (vs only `web` mode), and the change is identical in both
  files so symmetry is the only assurance.
- Consider folding the env-set into a tiny shared helper
  (`enableExperimentalFileWatcher()`) so the next handler that needs it
  doesn't copy-paste a magic string. Three sites is the threshold where
  the pattern starts being worth extracting.

## Verdict

`merge-after-nits` — the fix unblocks a real regression with the smallest
possible diff and matches existing prior art. Land it as-is for the
release, then file a follow-up to either flip the default or extract the
helper. Don't ship a fourth `process.env.OPENCODE_EXPERIMENTAL_FILEWATCHER = "true"`
in the codebase without addressing the design smell.
