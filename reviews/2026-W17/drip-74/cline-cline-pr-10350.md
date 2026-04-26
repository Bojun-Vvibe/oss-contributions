---
pr: 10350
repo: cline/cline
sha: e272a8dfbb02f1bab639545e977a33722717eb75
verdict: merge-after-nits
date: 2026-04-26
---

# cline/cline #10350 — docs: add SDK documentation

- **Author**: saoudrizwan (project maintainer)
- **Head SHA**: e272a8dfbb02f1bab639545e977a33722717eb75
- **Size**: +6926/-1387 across 58 files
- **State**: OPEN

## Scope

Three logically-distinct things bundled in one PR:

1. **New `docs/sdk/` tree (27 new `.mdx` pages)** — Mintlify documentation for the upcoming Cline SDK (`@clinebot/core`, `@clinebot/agents`, `@clinebot/llms`, `@clinebot/shared`). Organized into Getting Started (3) / Core Concepts (7) / Guides (8) / Architecture (3) / API Reference (6) sub-trees.
2. **`docs/cline-cli/` → `docs/cli/` rename** of all 13 existing CLI doc pages (12 `rename from/to` ops + 4 brand-new pages: `agent-teams.mdx`, `cli-reference.mdx`, `connectors.mdx`, `mcp-servers.mdx`, `scheduling.mdx`).
3. **Cross-doc link updates** in existing pages (`docs/api/*.mdx`, `docs/customization/hooks.mdx`, `docs/features/worktrees.mdx`, `docs/getting-started/*.mdx`, `docs/home.mdx`, `docs/mcp/mcp-overview.mdx`) to point `/cline-cli/...` → `/cli/...`, plus a top-level `docs.json` nav restructure to add the SDK tab.

The author also deletes `docs/cline-sdk/overview.md` (the old single-page SDK stub) — superseded by the new tree.

## Specific findings

- **Right call to bundle the rename + new content.** A reasonable instinct would be "split the rename into PR1 and the new content into PR2." But the new SDK pages link *into* the renamed CLI pages (e.g., `/cli/cli-reference#cline-auth`), and the rename leaves no compatibility redirects. Splitting would create a window where either (a) PR1 ships broken links from existing pages until PR2 lands, or (b) PR1 needs throwaway redirect shims. Doing both atomically is correct for a docs site that must be valid at every commit.

- **Link-update audit is the highest-risk surface and looks complete from the diff sample.** Every old `/cline-cli/...` reference I can see in the diff (e.g., `docs/api/authentication.mdx:69`, `docs/api/reference.mdx:208`, `docs/api/reference.mdx:251`, `docs/api/sdk-examples.mdx:217,245,272`, `docs/cli/acp-editor-integrations.mdx:201,205`) gets rewritten to `/cli/...`. This is the boring mechanical work that Mintlify's build-time link checker should catch — but the author should run `mintlify dev` (or whatever CI link-check exists) and confirm zero broken anchors before merge. The PR body admits `mintlify dev` failed locally on "pre-existing API doc parsing issues unrelated to SDK changes" — that's a yellow flag because it means the local link check was *not* run successfully. The Mintlify deployment preview will catch link breakage; reviewer should wait for that preview to pass before merging.

- **`docs.json` is the single-point-of-failure file.** A typo'd page reference there silently 404s in production. The PR body claims "Verified all 27 navigation page references in docs.json correspond to existing .mdx files" and "Validated docs.json is valid JSON" — both correct verifications, but reviewer should confirm by spot-checking 2-3 SDK nav entries against actual file paths in the diff. The 27 new files all live under `docs/sdk/...`; the nav entries should be `sdk/...` (Mintlify drops the `docs/` prefix).

- **No external backlink redirects.** The `/cline-cli/cli-reference` URL has presumably been live and indexed by search engines / referenced from blog posts / Stack Overflow answers / Slack channels. The rename to `/cli/cli-reference` will 404 every external link to the old path. **Mintlify supports redirects in `docs.json` via a `redirects` array** — strongly recommend adding entries for the 13 renamed pages:
  ```json
  "redirects": [
    { "source": "/cline-cli/cli-reference", "destination": "/cli/cli-reference" },
    { "source": "/cline-cli/overview", "destination": "/cli/overview" },
    ...
  ]
  ```
  This is a 13-line addition that prevents a measurable SEO hit and doesn't slow merge. Soft-block on this — the docs ship without it but with a quality-of-life regression.

- **27 new pages is a lot of surface area for typos / drift / inaccuracy.** The author's own caveat in "Additional Notes" is honest: *"Some type signatures and method names are based on the current state of the sdk-wip repo and may need updates as the SDK approaches release."* This means the docs will be partially-wrong at merge time and the team is committing to keep them in sync as the SDK stabilizes. That's a legitimate engineering choice (docs-as-design-surface), but it deserves a `<Note>` banner at the top of `docs/sdk/overview.mdx` saying *"The Cline SDK is in active development. APIs and types may change before the stable release; check the [changelog/release notes URL] for the latest signatures."* Otherwise users who copy-paste the example code will hit signature mismatches and file confused bug reports.

- **Removing the `cline-sdk/overview.md` stub is correct but leaves a redirect gap.** If anyone has linked to `/cline-sdk/overview` externally, that link 404s post-merge. Add a redirect there too:
  ```json
  { "source": "/cline-sdk/:slug*", "destination": "/sdk/overview" }
  ```

- **No tests, by design.** Documentation PRs don't need unit tests; the validation is "Mintlify build passes + manual page render check + link checker." The PR body lists the right verifications but explicitly admits one (`mintlify dev`) failed locally. A reviewer who waits for the Mintlify PR preview deployment to pass cleanly is doing the right gate.

- **Author is a project maintainer (`saoudrizwan`)** — owner-level commit, not first-time-contributor. Doesn't change the review bar but explains the scope.

- **Nit: the new `docs/cli/cli-reference.mdx` table at line 17-37 (Global Options) lists `--cwd <path>` but the existing `docs/cline-cli/getting-started.mdx` (renamed to `docs/cli/getting-started.mdx`) may have its own examples using positional args.** Reviewer should grep the renamed CLI pages for `--cwd` and `cd` usage to confirm consistency. Not a blocker; consistency is a follow-up.

- **Nit: progressive disclosure is the right pedagogical choice.** Getting Started → Core Concepts → Guides → Architecture → API Reference matches Stripe / Vercel / OpenAI docs as the PR body claims. The "Examples" page at the top of Getting Started (with Slack bot, code review agent, scheduled agent, CLI agent, extension plugin) is the right hook — concrete first, then concepts. Good IA.

- **Nit: 4 brand-new CLI pages (`agent-teams`, `cli-reference`, `connectors`, `mcp-servers`, `scheduling`) means 5 actually, not 4 as I implied above** — recount. Either way they're additive and unrelated to the rename, so they could have been a separate PR, but co-locating them with the `cline-cli` → `cli` reorg is acceptable since they live in the new directory.

## Risk

Medium. The risk profile is overwhelmingly **broken external links** rather than broken internal logic — internal links are mechanically rewritten and Mintlify will catch any miss in CI. External SEO impact and bookmarked-link breakage from the `/cline-cli/...` → `/cli/...` rename is real and irreversible without a redirects block.

The "27 pages may have stale type signatures" risk is acknowledged by the author and mitigated by the "active development" disclaimer suggestion above.

No code paths touched; rollback is `git revert` of one commit.

## Verdict

**merge-after-nits** — three soft asks before merge:
1. **Add a `redirects` block to `docs.json`** for the 13 renamed CLI pages and the deleted `cline-sdk/overview` page. Prevents external-link 404s.
2. **Confirm the Mintlify deployment preview passes the link checker** (since `mintlify dev` failed locally per PR body).
3. **Add a `<Note>` banner to `docs/sdk/overview.mdx`** (or `getting-started/overview.mdx` if that's the actual landing page) flagging that the SDK is in active development and signatures may drift before stable release.

The bundle-everything-atomically choice is correct given the cross-link dependencies. The documentation IA (progressive disclosure, examples-first) is well-shaped. The rename is the right cleanup.

## What I learned

The "rename without redirects" footgun is the docs equivalent of "delete a public API method without a deprecation cycle." Mintlify's `redirects` array is one of those features that's easy to forget exists until you've shipped a rename and started getting Discord pings about 404s. Worth a CI rule: any PR that contains a `rename from docs/.../foo.mdx to docs/.../bar.mdx` op should require a corresponding `redirects` entry in `docs.json` (or an explicit "no redirect needed" PR-body checkbox). Same rule applies to OpenAPI spec renames, frontend route changes, and any other URL-shaped public surface.
