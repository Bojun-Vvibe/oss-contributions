---
pr: 24533
repo: sst/opencode
sha: f6fd2f56d0ab2044795e4f3a26b27f6ad241f9e2
verdict: needs-discussion
date: 2026-04-27
---

# sst/opencode #24533 — docs: add comprehensive architecture documentation suite

- **Author**: jarek108
- **Head SHA**: f6fd2f56d0ab2044795e4f3a26b27f6ad241f9e2
- **Size**: +849/-0 across 8 new files under `docs/architecture/00…07_*.md`.

## Scope

Drops a 7-doc architecture suite into a new top-level `docs/architecture/` tree covering prompt composition, agents/modes, tooling/skills, execution loops, state/memory, security/config, and the Effect-TS + Pub/Sub framework. Pure markdown.

## Specific findings

- `00_architecture-overview.md:6-13` — TOC reads well. But "Reading Guide" mentions 7 components yet the index then numbers items 1–7 starting at "Ingestion & Context" (file `01_…`), so doc `00` is the meta-index and not part of the count — fine but worth a sentence.
- `01_prompt-and-context.md:5` — references `src/session/prompt.ts` and `src/session/llm.ts` via `../../packages/opencode/src/session/prompt.ts` relative links. Spot-checked the repo: `packages/opencode/src/session/prompt.ts` and `packages/opencode/src/session/message-v2.ts` do exist, so the link targets are real. `src/session/instruction.ts` referenced at `01_prompt-and-context.md:?` should be verified — that name doesn't appear in the current monorepo layout (instruction handling is in `session/system.ts` / `session/prompt.ts` last I looked), and stale path references are exactly what a docs PR like this drifts on at every refactor.
- `00_architecture-overview.md:34` — claims a "Fast Loop intercepts execution failures (e.g., catching a broken build or malformed artifact) and autonomously re-prompts the agent to fix it in the background." This describes an aspirational pattern more than what the current `session/prompt.ts` actually does — there's a synthesized-feedback loop for LSP/diagnostics but no general "broken build → re-prompt" daemon. Reads as marketing copy, not architecture doc. Maintainer should pressure-test every such claim against actual code paths before merge.
- `02_agents-and-modes.md:?` and `06_security-and-configuration.md:?` — descriptions of "Plan vs Build" mode "physically strips editing tools and injects explicit read-only constraints at the API level" — verify against `permission/` and `tool/` filtering code; the framing is louder than the implementation.
- `07_core-framework.md` — Effect-TS / `InstanceState` / Pub/Sub claims are accurate at the broad-stroke level, but no doc here is written from the perspective of "I just opened this repo, where do I look first?" — they're written as a marketing-shaped architecture brief. That's a different audience than what `docs/` typically serves in this repo.
- Zero attribution about how the suite was generated. 849 lines of polished prose dropped at once with the kind of mermaid+headings+pull-quote structure that's strongly suggestive of LLM authorship — fine if disclosed, problematic if not, because **stale-by-construction** docs are the single biggest tax this kind of contribution imposes on the repo. PR body just says "Verified formatting and content locally."

## Risk

Medium-high for a docs-only PR. Risk is not that it ships broken code — it's that it ships a 849-line "source of truth" surface that will silently drift the moment anyone refactors `session/prompt.ts`, and that the maintainer team now owns. Several claims read as confident but unverifiable, and at least one file path in the cross-references appears not to exist.

## Verdict

**needs-discussion** — maintainers should decide whether they want a `docs/architecture/` tree at all (it's a surface they'll have to maintain in lockstep with code), and if yes, every cross-reference to a `packages/opencode/src/...` path needs to be verified in the head SHA, every "the system does X" claim needs to be pressure-tested against the code, and the PR description should disclose authorship method. Not a code-quality issue — a maintenance-cost decision that wants explicit owner buy-in before merge.
