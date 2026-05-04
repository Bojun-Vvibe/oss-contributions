# sst/opencode #25714 — test(server): regression reproducers for #25698

- **Head SHA:** `b0aacad4eedbfbc5a481980650765b5d6d4704ca`
- **Size:** +105 / -0 across 2 files
- **Verdict:** **needs-discussion**

## Summary
Adds three regression tests in `packages/opencode/test/server/httpapi-listen.test.ts` and
`packages/opencode/test/server/httpapi-ui.test.ts` that pin the contracts behind cleanup
PR #25698 (PTY connect-token directory scoping + stripping upstream `transfer-encoding`
on proxied UI assets). Author flags the PR as deliberately red on `dev` until #25698
lands, and asks for either a rebase-after-merge or a fold-in.

## Notes
- The new `httpapi-listen.test.ts` case (around line 257) correctly asserts both halves
  of the directory contract: ambiguous mint → 404, scoped mint → 200, and a clean WS
  upgrade with the same `directory` query. That is exactly the right shape for the
  fix being landed in #25698.
- The `httpapi-ui.test.ts` case for `transfer-encoding: chunked` stripping is also
  well-targeted; using `serveUIEffect` directly with a stub `HttpClient` keeps the
  test deterministic without needing a real upstream.
- Test-only PR; no production code changes.

## Concerns
- Carrying a deliberately red PR violates the repo's preference for keeping `main`
  green and clouds CI signal for unrelated PRs. The author already acknowledges the
  preferred path is to fold these tests into #25698 itself.
- If #25698 changes shape during review, these tests will silently rot until #25698
  merges; the regression value drops the longer they sit.

## Recommendation
Fold these three tests into #25698 as a separate commit on that branch and close
this PR. If the maintainers prefer the test-after-fix sequence, hold this branch
until #25698 is queued and rebase right before merge — do not let it sit red on
`dev`.
