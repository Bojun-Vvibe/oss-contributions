# sst/opencode PR #25476 — Add Manifest provider docs section

- Head SHA: `ba65930a83daad421f0ed51451ba7ff5d3dfaff1`
- URL: https://github.com/sst/opencode/pull/25476
- Size: +55 / -0, 1 file (`packages/web/src/content/docs/providers.mdx`)
- Verdict: **merge-after-nits**

## What changes

Adds a new `### Manifest` section to the providers reference page,
between "OpenCode Zen" and "LLM Gateway". Documents the API key
prefix (`mnfst_`), the `/connect` + `/models` flow, and two config
shapes: cloud (no `api` URL) and self-hosted (Docker, defaulting to
`http://localhost:2099/v1`).

## What looks good

- Slot placement (line 9, between OpenCode Zen and LLM Gateway) is
  alphabetically sensible and matches the surrounding "router /
  gateway" cluster — a reader scanning for routing options will find
  it next to the comparable products.
- Both deployment modes are documented: the self-hosted JSON snippet
  (lines 49-60) is what differentiates this from the other router
  entries in the file. Worth keeping prominent.
- API key prefix (`mnfst_`) is called out at line 13 — useful for the
  future "redact secrets in logs" pass that other provider sections
  have already received.

## Nits

1. `https://manifest.build` (line 11) and the API-key bullet (line 13)
   make a marketing-flavored claim ("16+ providers", "smart routing
   across...") that the rest of providers.mdx has been actively pruned
   *out* of (compare the OpenCode Zen entry just above, which is now a
   single sentence). Suggest dropping the second sentence and keeping
   "Manifest is an open-source LLM router" as the bare description; the
   rest belongs on the upstream landing page, not in opencode docs.
2. The "Run the `/connect` command and search for Manifest" instruction
   (lines 15-19) implies `/connect` has fuzzy search built in. If it
   doesn't — i.e. if the user has to type the *exact* provider id — say
   so, otherwise this is the bug report magnet of the section.
3. The localized provider docs (`packages/web/src/content/docs/{ar,bs,
   da,de,es,fr,it,ja,ko,nb,pl,pt-br,ru,th,tr,zh-cn,zh-tw}/providers.mdx`,
   the same 18 locales touched in #25453) do **not** receive this entry.
   That's consistent with how other recent provider additions landed
   (English-first, l10n follows in a sweep), but worth either calling
   out in the PR body or filing a follow-up "translate Manifest section"
   issue so it doesn't quietly drift.
4. No anchor / id on the heading. The other provider sections in this
   file are linked from the dashboard ("how do I configure X"); a
   missing anchor here means inbound links from manifest.build can't
   deep-link to this section.

## Risk

Trivial — docs-only, single-file, no schema changes. The only failure
mode is the cloud config example showing `"manifest": {}` which assumes
the provider id `manifest` is registered in the schema; verify the
provider plugin actually ships with that exact id before merging,
otherwise users will paste this and get "unknown provider".
