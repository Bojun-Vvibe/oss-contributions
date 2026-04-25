---
pr: 24333
repo: sst/opencode
sha: d7ecfbbbaa96e55422a26b5577adefd4ae2a475a
verdict: needs-discussion
date: 2026-04-26
---

# sst/opencode#24333 — refactor: remove barrel index.ts and flatten export namespace

- **URL**: https://github.com/sst/opencode/pull/24333
- **Author**: alfredocristofano

## Summary

100-file sweep that (a) rewrites `AGENTS.md` from a short style guide
into a long onboarding doc, and (b) removes the `src/util/index.ts`
re-export barrel, replacing every `import { Log, Filesystem } from
"@/util"` style import with direct `import { Log } from
"@/util/log"`. Also flips `import { Config } from "@/config"` to
`@/config/config` in many files.

## Reviewable points

- The cross-cutting change pattern is consistent. Spot-check
  `packages/opencode/src/session/llm.ts` (the `Config` import path
  flip) and `packages/opencode/src/agent/agent.ts` (util barrel
  removal) — both rewrite imports without touching call sites, so
  TypeScript would catch any miss. Type-check is the actual gate.

- The repo's existing `AGENTS.md` carries explicit guidance:
  > In `src/config`, follow the existing self-export pattern at the
  > top of the file (for example `export * as ConfigAgent from
  > "./agent"`) when adding a new config module.
  This PR's new `AGENTS.md` deletes that line. Removing the barrel
  contradicts the documented pattern; either the doctrine should be
  updated explicitly ("we no longer use barrel re-exports") or this
  PR should be reframed as also a doctrine change. Right now it
  silently flips the convention.

- `@/util` was a stable public-feeling import surface across ~80
  files. Replacing it with N specific `@/util/<module>` imports
  doesn't yield a clear measurable win (no tree-shaking gain in a
  Bun-bundled CLI; no circular-dep elimination called out). The
  cost is that future `@/util` consumers need to know which
  submodule each helper lives in. A short justification in the PR
  description would help reviewers decide if the trade is worth it.

- Renaming `import { Config } from "@/config"` → `@/config/config`
  is awkward (`config/config`) and suggests the *new* convention
  also wants more thought — maybe `@/config/index` was fine, or
  maybe the right move is to rename the file to `@/config.ts`
  flat. This PR commits to the awkward intermediate.

- The `AGENTS.md` rewrite is mostly additive and useful (Bun
  version pin, monorepo map, dev commands). It also drops some
  prior style rules ("Avoid `try`/`catch` where possible", "Reduce
  total variable count by inlining when a value is only used
  once"). Those drops should be intentional, not collateral.

## Rationale

Mechanical rename across 100 files is reviewable in isolation, but
the convention flip + `AGENTS.md` rewrite + dropped style rules are
three policy changes bundled into one refactor PR. Maintainer
should weigh in on whether the barrel removal is the new
convention before this lands; otherwise the next contributor will
re-add a barrel and trigger the same churn in reverse.

## What I learned

A "refactor: flatten exports" PR that *also* rewrites the project's
authoritative style guide is two PRs in one. The mechanical
diff-checking is easy; the convention question is the actual
review surface and is exactly what gets lost in a 3000-line patch.
