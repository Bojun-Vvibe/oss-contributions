# anomalyco/opencode PR #24220 — Fix session event typechecks and shell cwd

- **Repo:** anomalyco/opencode
- **PR:** [#24220](https://github.com/anomalyco/opencode/pull/24220)
- **Head SHA:** `81d81e63341ac1831126ab2cab1c9073fca31cd6`
- **Author:** thdxr
- **Size:** +927 / −466 across 8 files
- **Reviewer:** Bojun (drip-27)

## Summary

Three loosely-related fixes shipped together:

1. Keep shell commands anchored to the intended instance cwd after
   a login shell's startup files run (which can `cd` away).
2. Add required session IDs to v2 session events and a fixed
   `SessionID.empty()` helper for tests.
3. Register v2 session-event sync definitions before the sync
   registry freezes.

Verification per PR body: `bun typecheck` from `packages/opencode`,
plus targeted tests at `test/session/session-entry-stepper.test.ts`
and `test/session/prompt.test.ts`.

## Key changes

### Shell cwd: `packages/opencode/src/session/prompt.ts:795–833`

The shell wrapper used to capture `__oc_cwd=$PWD` *after* sourcing
`/etc/profile` and the user's `~/.bashrc` / `~/.zshrc`. If any of
those `cd`'d (common in dotfiles that auto-activate Python/node
envs based on directory), the captured `$PWD` was wrong and every
subsequent `cd "$__oc_cwd"` snapped the shell into the dotfile's
preferred directory rather than the instance cwd opencode handed in.

The fix:

```diff
-              __oc_cwd=$PWD
+              __oc_cwd=$OPENCODE_CWD
```

…in two of the three command branches (login and non-login), with the
matching env-var injection at the spawn site:

```diff
-        env: { ...shellEnv.env, TERM: "dumb" },
+        env: { ...shellEnv.env, OPENCODE_CWD: cwd, TERM: "dumb" },
```

Threading `cwd` through an env var that the wrapper reads is the
right approach — it's immune to any `cd` the dotfiles do, and it
doesn't require the wrapper to know how to escape the path safely
into the script body.

### Permission config order: `packages/opencode/src/config/permission.ts:1–80`

Replaces the old `__originalKeys` preprocess hack with a parallel
zod schema (`InfoZod`) attached via the `ZodOverride` annotation.
The new `InfoZod` parses user input as a union of `Action |
Record<string, Action | Record<string, Action>>`, normalizes via
`normalizeInput`, then `superRefine`s to enforce that the
`ACTION_ONLY` keys (`todowrite`, `question`, `webfetch`,
`websearch`, `codesearch`, `doom_loop`) can't carry per-pattern
sub-objects.

Notably the doc comment now says "Runtime config parsing uses
`InfoZod` below so user key order is preserved for permission
precedence" — the `fromConfig` call site previously sorted top-level
keys to make wildcard precedence order-independent, but the comment
implies that sort is now *not* relied upon. Worth confirming this
in the diff for `Permission.fromConfig` (which isn't in this PR's
file list, so presumably it landed in an earlier commit).

### v2 session events: `packages/opencode/src/v2/session-event.ts`

Net `+82 / −138` reshape of the file (per the hunk count). The
`SessionEvent` namespace now requires session IDs on event
constructors that previously made them optional. Combined with
`SessionID.empty()` as a fixed test sentinel, this lets the test
suite construct events with a deterministic-but-clearly-not-real
session ID instead of `undefined` or random strings.

### Sync registration: `packages/opencode/src/session/projectors.ts:5`

```diff
+import "../v2/session-event"
```

A bare side-effect import. The purpose is to ensure
`v2/session-event.ts` runs its registry-registration calls *before*
`session/projectors.ts` triggers the sync registry to freeze. Order
of imports here is load-bearing — removing this import would
silently break sync emission for v2 events.

### `packages/opencode/src/session/schema.ts:8–10`

```diff
 export const SessionID = Schema.String.annotate({ [ZodOverride]: Identifier.schema(...) })
+export namespace SessionID {
+  export const empty = ...
+}
```

`SessionID.empty()` constructor for tests. Keeps test code from
hand-rolling fake session IDs that drift from the real schema's
constraints.

### Test updates: `test/session/session-entry-stepper.test.ts`

Adapts existing tests to thread `SessionID.empty()` through the
event constructors that newly require an ID. ~20-line delta to
add the IDs in the right places.

## What's good

- The `OPENCODE_CWD` env-var approach to anchoring the shell cwd is
  exactly right — it survives any `cd` the user's dotfiles do, and
  it's transparent to the shell (the env var is just data, not
  interpreted code, so no escaping concerns). Much better than the
  alternative of disabling dotfile sourcing entirely (which would
  break every user who relies on `nvm`/`pyenv`/`direnv` activation).

- Replacing the `__originalKeys` preprocess hack with a proper
  zod-side override via `ZodOverride` annotation is a real
  improvement to the permission config story. The comment block
  on `permission.ts:20-29` is honest about *why* the hack existed
  ("`Permission.fromConfig` depended on the user's insertion
  order") and *why* it's no longer needed ("`fromConfig` now sorts
  top-level keys").

- `ACTION_ONLY` set + `superRefine` on the `InfoZod` is the right
  shape for "these keys must not carry sub-patterns". Catches the
  `todowrite: { "*": "allow" }` typo at config-parse time rather
  than at first invocation.

- The bare `import "../v2/session-event"` in `projectors.ts` is
  load-bearing in a way that's easy to lose in a refactor. Worth
  flagging via comment (see concern 2).

## Concerns

1. The shell-cwd fix only updates two `__oc_cwd=$PWD` sites in the
   diff (lines 82 and 91). If there are other branches in the
   wrapper that I didn't see in the truncated diff (e.g. a
   non-login non-interactive branch), they may still capture
   `$PWD` post-rc. Worth grepping the file for any remaining
   `__oc_cwd=$PWD` references before merge.

2. `import "../v2/session-event"` in `projectors.ts:5` has no
   inline comment explaining *why* it's a bare import. A future
   contributor running an "unused import" lint or doing an import-
   sort pass could remove it and silently break sync emission.
   One-line comment:

   ```ts
   // side-effect import: registers v2 session-event sync definitions
   // before the sync registry freezes (must precede projector use).
   import "../v2/session-event"
   ```

3. `SessionID.empty()` being a fixed sentinel rather than
   `SessionID.random()` per call is the right ergonomic for tests
   (deterministic IDs make snapshot tests stable), but the
   underlying value should be one that's *obviously not real* —
   `00000000-0000-0000-0000-000000000000` or similar. If it's just
   an empty string or a default-zero, accidentally leaking it into
   prod data would be hard to tell apart from a missing field.
   Worth verifying the actual value chosen.

4. The `+927 / −466` is dominated by the v2 session-event reshape
   (`+82 / −138`-ish in that file alone, plus the full re-export
   surface in `sdk/js/src/v2/gen/types.gen.ts` getting regenerated).
   Reviewers should be aware that a substantial chunk of the diff is
   generated SDK code, not handwritten — so the actual review surface
   is closer to ~300 lines of session-event.ts rewrite plus the four
   smaller fixes.

5. Concern about the doc-comment claim "fromConfig now sorts
   top-level keys so wildcard permissions come before specifics":
   this PR's diff does *not* touch `fromConfig`. Worth confirming
   that sort lives in `Permission.fromConfig` already (presumably
   from a prior commit) and that this PR's `InfoZod` doesn't
   re-introduce a key-order dependency that the sort would mask.

## Risk

Moderate. Three independent changes in one PR raises the bisect-
difficulty if any of them regresses. The shell-cwd change is the
highest risk (every shell invocation goes through it), but also
the most thoroughly justified — the existing behaviour was
demonstrably broken for users with `cd`-on-startup dotfiles.

## Verdict

**merge-after-nits**

The fixes are individually well-motivated and the implementations
are sound. Concern 2 (comment on the side-effect import) is the one
thing I'd want before merge — bare imports without comments are
landmines for future refactors. Concerns 1, 3, and 5 are quick
verifications the author should be able to confirm with a grep
or a glance at the related file.

## What I learned

The pattern of "thread cwd through an env var the wrapper reads"
is a clean general solution to "user dotfiles can `cd` and break
my assumed working directory" — much better than disabling rc
files (breaks user envs) or capturing `$PWD` post-rc (broken in
this exact case). Generalises to any context where you need to
spawn an interactive shell but care about the post-init working
directory: `git`, `tmux`, `vscode`'s integrated terminal, etc.,
all hit this and all solve it via roughly the same env-var-passing
trick.
