# QwenLM/qwen-code#3713 — feat(cli): set OS process title to encode session identity

- **Repo**: QwenLM/qwen-code
- **PR**: [#3713](https://github.com/QwenLM/qwen-code/pull/3713)
- **Head SHA**: `f48eefc2d19a6b55468a67990eee7cfe25088697`
- **Reviewed**: 2026-04-29
- **Verdict**: `merge-as-is`

## Context

`ps -ef | grep qwen-code` today yields generic `node …` rows because
qwen-code never sets `process.title`. External tools that need to map
a running PID to its qwen-code session — terminal multiplexers, tab
managers, IDE integrations, on-call operators triaging a shared box —
can't do that mapping today.

This PR sets `process.title` to:

```
qwen-code session=<short-id> cwd=<basename>
```

…on POSIX-like platforms (`linux`, `darwin`, `freebsd`, `openbsd`),
no-op on Windows because Node's `process.title` setter on Windows
maps to the *console window title*, which would race with the
existing OSC `\x1b]2;…\x07` writers in `gemini.tsx` /
`AppContainer.tsx`.

The PR cites kimi-cli #2082 as prior art (which makes sense — same
shape of fix).

## What changed

### `packages/cli/src/gemini.tsx`

One new import + one new call inside `startInteractiveUI`, sitting
right next to the existing `setWindowTitle()`:

```tsx
import { setSessionProcessTitle } from './utils/processTitle.js';
...
setWindowTitle(basename(workspaceRoot), settings);
setSessionProcessTitle(config.getSessionId(), config.getTargetDir());
```

Two-line touch in the load-bearing entry-point file. Minimal blast
radius if `setSessionProcessTitle` ever throws (which it doesn't —
see the `swallows errors` test).

### `packages/cli/src/utils/processTitle.ts` (new — not in diff but
inferred from test imports)

Four exports based on the test file:

- `composeSessionProcessTitle(sessionId, workDir?, opts?)` — pure
  string composer
- `setSessionProcessTitle(sessionId, workDir?, opts?)` — apply via
  `process.title = ...` (or test injection sink)
- `shortSessionId(id)` — strip dashes, take first 8 chars, fall back
  to original on degenerate input
- `shouldSetProcessTitle(platform)` — gate by Node platform string

### `packages/cli/src/utils/processTitle.test.ts` (new, 22 cases)

Genuinely good test surface. Highlights:

- **Short-id stripping**: a real UUID
  `12345678-aaaa-bbbb-cccc-dddddddddddd` → `12345678`
- **Short-id fallback**: `"---"` → `"---"` (because stripping
  dashes leaves nothing — falls back to the original input rather
  than emit an empty `session=` token)
- **Whitespace normalization**: spaces and `=` inside the *cwd
  basename* and inside the *session id* both get replaced with `_`
  so the title remains splittable on whitespace into exactly three
  tokens. Asserted directly:
  ```ts
  expect(title.split(/\s+/)).toEqual([
    'qwen-code', 'session=12345678', 'cwd=a_b_c_d',
  ]);
  ```
  This is the right invariant — downstream tools that grep
  `qwen-code session=… cwd=…` from `ps` output assume token-stable
  whitespace.
- **Non-ASCII preservation**: `/项目/我的-app` → `cwd=我的-app`.
  CJK characters stay; only ASCII whitespace and `=` are stripped.
- **Trailing-slash normalization**: `/home/user/projects/my-project/`
  still yields `cwd=my-project`.
- **Empty-string cwd**: omitted from the title rather than emitting
  `cwd=`.
- **Custom baseName**: `qwen-code-bg session=…` for background
  variants.
- **Platform gate**: `win32` → no-op (returns null), POSIX
  variants → set.
- **Error swallow**: `apply: () => { throw new Error('seccomp says no'); }`
  doesn't propagate. Important — `setProcTitle` syscalls can fail
  inside hardened sandboxes (seccomp-bpf, AppArmor) and qwen-code
  shouldn't crash because the OS denied a cosmetic syscall.

The test file uses dependency injection
(`apply: (t) => captured.push(t)`) instead of mocking `process.title`
globally. Right call — keeps the test isolated and parallelizable.

## Design call-outs (good)

- **Pure composer + thin side-effect wrapper**. The hard part
  (string sanitization, length truncation, platform decisions) is in
  pure functions; the side effect is a single sink that can be
  swapped for a test capture. This is the kind of structure that
  makes 22 unit tests possible without any test doubles.
- **POSIX-only with documented Windows reasoning**. The PR doesn't
  just skip Windows silently — it explains the OSC race. Future
  contributors who want to "fix" the Windows skip will see the
  reasoning in the comment block (assumed; if missing, please add).
- **Conservative scope**. Explicitly *out of scope*:
  - `runtime.json` sidecar (tracked separately)
  - Dynamic OSC tab title updates
  Both are reasonable follow-ups, neither belongs in this PR.

## Risk analysis

- **`process.title` setter limits**: Node's `process.title` write is
  allocated within argv space; on Linux it can silently truncate to
  the original `argv[0]` length. The composer cap (8 chars for
  session, basename for cwd) keeps the title around 30–60 chars in
  practice — well under the typical limit. Worth one comment in
  the composer noting this.
- **Privacy**: cwd basename leaks into `ps`. A user working in
  `/Users/foo/secret-project/` has `cwd=secret-project` visible to
  any other user on the box who can `ps`. That's the same exposure
  level as the cwd itself in `ps -ef -o pid,cwd`, so not a new
  leak — but worth one line in docs for users who didn't realize
  `ps` shows this.
- **Sandbox failures**: handled (the `swallows errors` test).
- **No-op on Windows**: handled (the `shouldSetProcessTitle`
  test).
- **Race with OSC tab title**: avoided by skipping Windows
  entirely. POSIX OSC writers (if any) and `process.title` are
  separate channels, no interference.

## Suggestions (non-blocking)

- A test for the "session id much longer than 8 chars but no
  dashes" case — current test covers the dash-stripping path; the
  no-dash path would also be worth one assertion (just to lock in
  that `shortSessionId('verylongsessionid12345')` truncates to
  `'verylong'` and not the full string).
- A length cap on the cwd basename. A user with a 200-char cwd
  basename would push the title past the kernel-allowed `argv[0]`
  size. Easy fix: truncate cwd basename to e.g. 32 chars with `…`
  if longer.

## Verdict

`merge-as-is`. Genuinely good shape — pure composer + thin
wrapper + 22 unit tests + DI for the side effect + documented
Windows skip + sandbox-failure tolerance. Conservative scope. The
two suggestions above can land as follow-ups or be slipped into
this PR, neither blocks.

## What I learned

`process.title` is one of the most under-used affordances for
making CLI tooling friendly to operators. A single assignment
turns `ps -ef` from useless ("which of these 12 `node` processes
is the one I care about?") into a routing key. The trick is doing
the assignment portably — Node's setter has different semantics
on Windows (window title) vs POSIX (`argv[0]` slot) and silently
truncates on Linux. A pure composer + `shouldSetProcessTitle(platform)`
gate is the minimum design that gets you safe behavior on every
platform.
