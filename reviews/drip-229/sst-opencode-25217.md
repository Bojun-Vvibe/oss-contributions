# sst/opencode #25217 — test(httpapi): cover more safe GET parity

- **PR**: https://github.com/sst/opencode/pull/25217
- **Head SHA**: `1468513da9e2`
- **Files reviewed**: `test/server/httpapi-json-parity.test.ts`
- **Date**: 2026-05-01 (drip-229)

## Context

Tests-only follow-up to the `Schema.optional → optionalOmitUndefined`
contract flip from #25214. That earlier PR realigned the HttpApi
backend's JSON encoding with the legacy backend's "absent means key
omitted" shape, validated by the `httpapi-json-parity.test.ts`
endpoint-by-endpoint sweep. This PR broadens the sweep to cover seven
more safe-GET endpoints that were not in the original parity matrix.

## Diff (1 file, +7 -0)

`test/server/httpapi-json-parity.test.ts:9-12,124,140-141,144,171`:

New entries appended to the sweep table:

```diff
+ import { PtyPaths } from "../../src/server/routes/instance/httpapi/groups/pty"
  ...
+ { label: "global.config", path: GlobalPaths.config, headers: {} },
  ...
+ { label: "permission.list", path: "/permission", headers },
+ { label: "question.list", path: "/question", headers },
+ { label: "pty.shells", path: PtyPaths.shells, headers },
+ { label: "pty.list", path: PtyPaths.list, headers },
  ...
+ { label: "experimental.resource", path: ExperimentalPaths.resource, headers },
```

Each entry is fed to `expectJsonParity({ ...input, legacy, httpapi })`
under `concurrency: 1`, which already exists from #25214 and asserts
byte-for-byte JSON equality between the legacy and HttpApi backends for
the given path.

## Observations

1. **Correct lift via the central runner.** The seven new cases inherit
   parity coverage for free: `expectJsonParity` is the source-of-truth
   comparator from #25214 and the helper does not need to change. This
   is the right shape for a parity-table extension.

2. **Sensible target selection.** The added endpoints span four
   semantically distinct shapes:
   - `global.config` — top-level singleton object (likely the most
     `optionalOmitUndefined`-sensitive of the batch because user
     config has many absent-vs-null fields).
   - `permission.list` / `question.list` — collection endpoints
     (variable-length arrays, optional per-item fields).
   - `pty.shells` / `pty.list` — system-discovered + session-scoped
     collections (different optional-field profile from
     permission/question).
   - `experimental.resource` — experimental surface that has been a
     drift hotspot historically and benefits from explicit pinning.

3. **No fixture surface change.** The test bootstrap (legacy + httpapi
   in-process server, `concurrency: 1` to avoid cross-test bleed,
   shared `headers`) is reused untouched. Reviewers can be confident no
   incidental contract change is hiding in this diff.

## Nits

- **`pty.list` may be empty in CI.** If the test environment has no
  active PTY sessions, `pty.list` returns `[]` from both backends and
  the parity check trivially passes — it would not catch a regression
  in the populated-list shape. Worth a follow-up arm that seeds at
  least one PTY before the assertion (or a comment noting the
  intentional empty-state coverage).

- **No `permission.list` populated arm.** Same shape as above:
  empty-list parity is a weaker contract than "list of N items, each
  with absent optional `denied_at` etc., still parity-equal." Consider
  one populated arm per collection endpoint as a follow-up.

- **`experimental.resource` is undocumented in the test.** The other
  `experimental.*` arms are alongside in the table; a one-line
  comment ("experimental surfaces drift fastest, pin them
  explicitly") would survive a future "this is experimental, why are
  we testing it" trim attempt.

## Verdict

`merge-after-nits` — pure test-only coverage extension at the parity
matrix, no behavior change, low risk. Nits are populated-arm coverage
for the two collection endpoints and a one-line rationale comment on
the experimental entry.
