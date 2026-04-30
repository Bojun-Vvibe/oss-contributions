---
pr-url: https://github.com/BerriAI/litellm/pull/26802
sha: 466ce9dfb65b
verdict: merge-after-nits
---

# Lazy loaded imports, lazy loaded front page

Big one — `+32608/-106` — but the actual logic is small (~600 LOC across `_lazy_features.py` and `_lazy_openapi_snapshot.py`); the bulk is the 31651-line generated `_lazy_openapi_snapshot.json`. The premise is concrete: deployments that don't use vector-store / batches / managed-files / Workflow-Runs etc. were paying ~700 MB resident memory on import for routers they never call. The fix introduces `LAZY_FEATURES` at `litellm/proxy/_lazy_features.py:1-432` — each entry pairs a path-prefix with an import callable that runs on the *first* request matching that prefix.

The accompanying CI workflow at `.github/workflows/check-lazy-openapi-snapshot.yml:1-75` is the right shape: it regenerates the snapshot to `/tmp`, diffs against the committed copy, and on drift uses `LouisBrunner/checks-action` to mark the check **neutral** (not failing) — so a stale snapshot doesn't block PRs but does surface visibly. The body explicitly says "the snapshot will regenerate at release if not committed," which is the right fail-soft policy because the snapshot is a derivative artifact.

Nits: (1) the `concurrency` group correctly uses `cancel-in-progress: true` for PR runs, but the snapshot itself contains `_lazy_openapi_snapshot.json` (31651 lines) committed to the repo — every PR that touches a route handler will produce an unrelated noisy diff there, which will train reviewers to ignore the snapshot churn (the CI's neutral-on-drift policy partially mitigates this but the bot-comment UX is missing); (2) `_include_router` and `_mount_app` in `_lazy_features.py:32-46` close over `attr_name` / `prefix` via `Callable` factories — fine, but no test pins that calling the same registration twice (e.g. via concurrent first-requests racing) is idempotent under `asyncio` — the file imports `asyncio` so a per-feature `asyncio.Lock` would be the safer shape; (3) first-hit latency cost (1-3s for heavy modules per the docstring) is not surfaced in any user-facing log line at registration completion.

## what I learned
A 31k-line generated snapshot committed to the repo is acceptable iff the CI policy explicitly tolerates drift (neutral, not failing) and the regeneration is a one-line `python -m ...` command — both true here. The wrong shape would be making the check blocking, which would convert every router-touching PR into a "rebase + regenerate snapshot" ritual.
