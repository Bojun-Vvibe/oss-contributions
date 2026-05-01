# sst/opencode#25166 — docs: add missing docs for global config endpoints

- Repo: sst/opencode
- PR: https://github.com/sst/opencode/pull/25166
- Head SHA: `964c32b731d4`
- Size: +8 / -6 (one file)
- Verdict: **merge-as-is**

## What the diff actually does

Single file: `packages/web/src/content/docs/server.mdx`.

1. Extends the **Global** endpoint table (lines 89–94 of the new file) with two
   previously undocumented rows:
   - `GET  /global/config` → `Config` type
   - `PATCH /global/config` → `Config` type
   The existing `/global/health` and `/global/event` rows are unchanged; the
   header/separator widths bump from 6 to 7 dashes so the column widths still
   align — the only "code" change in the diff is that markdown re-flow.
2. In the **Config** section (line 128 area) renames the two existing
   `/config` rows from "Get config info" / "Update config" to "Get instance
   config info" / "Update instance config", which is the load-bearing wording
   change: it disambiguates instance-scoped config from the new
   global-scoped config that this PR is documenting.

## Why merge-as-is

- **Correctness check on the new rows:** the route shapes (`GET` and `PATCH`
  on `/global/config` returning `Config`) match the existing instance-scope
  pair, which is the right symmetry for an MDX docs page that derives its
  link target from `<a href={typesUrl}>` shared across both blocks.
- **Disambiguation is the right scope.** Without the "instance" wording flip
  on the old rows, a reader landing on the page would not be able to tell why
  there are now two `/config` and two `/global/config` rows — the rename
  closes that ambiguity in one word per row, with no behavior change.
- **PR author verified by exercising both `PATCH /config` and
  `PATCH /global/config`** against a running server (per PR body), so the
  documented shape is grounded, not inferred.
- Pure docs PR, one MDX file, no code paths touched, no risk surface.

## Nits I would NOT block on

- The new column-width bump (6→7 dashes in the separator row) is a noisy
  diff line that adds nothing — a future maintainer reading `git blame` will
  see the docs row addition and the alignment churn together. Pre-merge
  squash already collapses this, so it's invisible to the next reader.

## Theme tie-in

Doc-as-spec for an undocumented surface that already shipped: the API
existed, callers were already using it (PR author in particular), the docs
were the only thing missing. Closes the spec gap at the docs layer rather
than at every consumer's "I had to read the source" workaround.
