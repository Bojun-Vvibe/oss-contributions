# block/goose #9028 — docs: improve goose2 AGENTS.md

- **Head:** `3dc3e7d` (`3dc3e7d288caa419ac4903d3c9e43af5f3ea1899`)
- **Repo:** block/goose
- **Scope:** `ui/goose2/AGENTS.md` only (~60 net lines, mostly restructure + new "Don't build features entirely in the frontend" section)

## What the diff does

Three logical edits to the goose2 frontend AGENTS.md:

1. **Promotes the "frontend → ACP → goose core" architecture diagram from a buried section to right after the build-quickstart block** (`AGENTS.md:32-46` in the new layout). The diagram itself gets a small but meaningful upgrade: the chain now includes `features/<feature>/api/` as an explicit hop, making it clear that the SDK is *not* called directly from UI code.

2. **Adds a new section "Don't build features entirely in the frontend"** (`:48-60`) that names 5 specific anti-patterns:
   - Business logic in a Zustand store / React hook / api adapter that calls `localStorage`, `fetch`, or filesystem
   - `invoke()` proxying to goose instead of an ACP custom method
   - Persisting domain state to `localStorage` (with a clarification that `localStorage` IS appropriate for theme mode, sidebar collapse, last tab)
   - Manually duplicating types between TS and Rust instead of using the generated SDK

3. **Updates the feature-organization table at `:91-101`** to add a "Backend shape" column, mapping each pattern (Full-featured / Data-driven / API features / Simple features / Tabs) to its expected backend interaction shape (typed ACP / CRUD via ACP / Pure pass-through / No backend / No backend). Adds an enforcement-rule line at `:107` saying only `features/<feature>/api/` and `shared/api/` may import from `@aaif/goose-sdk` or call `getClient()`.

4. **Drops a redundant duplicate "## Architecture" block** (the old buried section at the end of the file is removed since it's now the lead).

## Strengths

- **The "Don't build features entirely in the frontend" section names the failure mode by example, not by abstract principle.** The 5 specific anti-patterns map directly to mistakes someone vibing with Cursor / agentic coding tools is most likely to make ("just throw it in localStorage", "just add an invoke()"). That's the right frame for an `AGENTS.md` whose audience is part-LLM.
- **`localStorage` carve-out is well-chosen** — explicitly listing theme mode, sidebar collapse, and tab state prevents the over-correction where someone moves UI ephemera into goose core and clutters the config surface.
- **The "Backend shape" column on the feature table** turns a previously-implicit mapping into an explicit contract. Makes it grep-able when reviewing a new feature PR — "the table says Data-driven means CRUD via ACP, where's your ACP method?".
- **Cross-reference to PR #8675 ("skills-as-sources")** as the canonical example is already in the file (preserved at `:111+`); the new lead section now points at it explicitly via "see the canonical example below".

## Concerns

1. **Two slightly conflicting statements about `Full-featured` features** — old text said `Full-featured = stores + hooks + ui`, new text says `Full-featured = stores + hooks + ui + api`. The "+ api" addition is intentional and correct (full-featured features that don't have backend calls don't exist as Full-featured by definition), but if any existing feature directory in `goose2/src/features/` is currently in the old `Full-featured` shape without `api/`, this PR's table will mis-classify it. Worth grepping the tree before merge.
2. **No git-blame anchor on the "PR #8675" reference** — if that PR ever gets squash-merged with a different SHA, the doc reference still works (PR numbers are stable), but linking to the specific files (`features/skills/api/skillsApi.ts` etc.) would survive a renumber and be more useful.
3. **Pure docs PR with no code change** — risk surface is essentially zero. The only failure mode is contradicting an existing rule somewhere else in the repo; quick `grep -r "Architecture"` of `AGENTS.md` files in `ui/goose2/**` would catch that.

## Verdict

**merge-as-is** — high-quality docs hardening that codifies an architectural rule the project clearly cares about. The "don't build features entirely in the frontend" section is the kind of explicit-rather-than-implicit guidance that pays for itself the first time a new contributor (human or agent) avoids a `localStorage`-as-backend mistake. Head `3dc3e7d`.
