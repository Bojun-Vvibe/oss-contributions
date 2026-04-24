# sst/opencode#24047 — docs: add agent architecture audit guide

- **PR:** https://github.com/sst/opencode/pull/24047
- **Head SHA:** (single-file PR, AGENT-AUDIT.md)
- **Files:** `AGENT-AUDIT.md` (+144/-0)
- **Verdict:** **request-changes**

## Context

A docs-only PR adding a single new top-level `AGENT-AUDIT.md`
describing a "12-layer agent stack" with grep commands, a severity
model, and a JSON report template. The author's stated goal is "help
opencode users audit their own agent systems for hidden failures" —
hardcoded secrets, unrestricted code execution, missing tool
enforcement, unbounded memory growth.

The PR body itself flags the lift: *"The full audit logic lives in a
companion Python library (agchk, https://github.com/.../agchk) which
is published on PyPI. This document references it and adapts the
guidance to the OpenCode developer experience."* The PR does not
actually link to that library inside the file — the only outbound
links in the diff are to `https://opencode.ai/docs`,
`./packages/plugin/`, and `./CONTRIBUTING.md`.

## Problem

Three load-bearing problems make this not-mergeable as written:

1. **The doc is content-marketing scaffolding, not opencode-specific
   guidance.** Every section that claims to be "for opencode" is
   actually framework-agnostic. The "12-Layer Stack" table at lines
   25–40 of the new file lists generic layers (`System prompt`,
   `Session history`, `Long-term memory`, `Distillation`, `Active
   recall`, …) with no reference to any actual opencode surface — no
   `Session.compact`, no `Provider.transform`, no `Plugin.hooks`, no
   `tool/registry.ts`, no `acp/agent.ts`. A reader who knows opencode
   gets nothing actionable; a reader who doesn't gets a cargo-cult
   checklist they can't execute.

2. **The grep recipes are wrong for the opencode tree.** The "Quick
   Diagnostic" block recommends:

   ```
   rg "must.*tool|required.*call|必须.*工具" --type md --type ts
   rg "tool_call|toolCall|tool_use" --type ts
   rg "completion|chat\.create|messages\.create" --type ts
   ```

   These patterns are pulled from the agchk library's defaults and
   match almost nothing useful in the opencode TS sources. The
   AI-SDK-based tool path uses `tools: { … }` with a Zod schema and
   `Tool.execute`; provider call sites go through `streamText` /
   `generateText`, not `chat.create` / `messages.create`. The mixed
   English/Chinese alternation `必须.*工具` is also a giveaway that
   this regex was copied from an unrelated codebase. Telling opencode
   users to grep their own plugins for `messages.create` will produce
   silent zeros and they'll think they passed the audit.

3. **The "Report Template" + severity table promote a workflow the
   project does not endorse.** Lines 99–138 of the new file ask the
   reader to emit a structured `findings[]` JSON with
   `severity: critical|high|medium|low`, `source_layer`,
   `mechanism`, `root_cause`, `recommended_fix` and an
   `ordered_fix_plan`. That's a *product surface* — it implies an
   external tool is going to consume this JSON. The PR body confirms
   the consumer is `agchk`. Adding a doc to opencode's repo whose
   primary purpose is to feed an unrelated PyPI package crosses the
   line from documentation into upstream-funnel for a side project.

## Smaller issues

- The doc lives at the repo root (`AGENT-AUDIT.md`), not under
  `docs/` or `packages/web/src/content/docs/`. Other top-level docs
  (`README.md`, `CONTRIBUTING.md`, `LICENSE`) are repo-meta; this
  doesn't fit. If it stays, it belongs as an mdx page in the docs
  site so it picks up the existing nav and search.
- "Common Failure Patterns → Wrapper Regression" claims *"The base
  model works fine via `opencode serve` API, but your wrapper agent
  breaks it."* — there is no `opencode serve` subcommand exposed at
  the user CLI level today; the relevant entry point is `opencode
  start --port`. Bad doc references date instantly.
- The "Memory Contamination" check tells readers to inspect
  `session.json` — but opencode's persistent state is keyed off
  `~/.local/share/opencode/project/<project_id>/storage/` with
  multi-file storage, not a single `session.json`. This is just
  wrong.

## Risks

- Future agents grepping this file for "how does opencode handle
  tool calls?" will be misled by the framework-agnostic 12-layer
  table.
- Linking from the project's own repo lends implicit endorsement to
  the third-party `agchk` package. If `agchk` later breaks, gets
  taken over, or starts shipping telemetry, the doc becomes a
  liability.
- The "must.*tool" regex (in mixed Chinese/English) is the kind of
  thing that gets later "fixed" by a well-meaning contributor into
  something that *actually* matches opencode source — at which point
  the doc becomes load-bearing for behavior it was never designed
  to describe.

## Suggestions

If the maintainers want a doc here at all, the high-value version
would be much smaller and much more concrete:

1. A page under `packages/web/src/content/docs/plugins/` titled
   "Auditing your plugin" that points at the *actual* enforcement
   seams: `Tool.execute` validation, `Permission.ask`, the
   `Provider.transform` chain, hook ordering in `Plugin.hooks`, and
   the `Session.compact` boundary.
2. Real `rg` recipes that match the TS sources — e.g. *"find tools
   that don't validate inputs against their schema before
   execution"*, with a regex that actually catches `Tool.execute`
   handlers missing a `parse(input)` call.
3. Drop the `agchk` framing entirely. If the author wants to publish
   audit logic, that belongs in `agchk`'s own README, with a link
   *into* opencode docs from there, not the reverse.

## Verdict — request-changes

Documentation PRs deserve a high bar precisely because they're
cheap to land and expensive to remove. This one is an external
checklist re-skinned with the project's name and bolted onto the
repo root; it doesn't help the opencode user, it confuses them. The
right move is to close in favor of either (a) a much smaller
opencode-specific plugin-auditing page, or (b) a link from
`agchk`'s own docs into existing opencode plugin docs.

## What I learned

The pattern of *"docs PR that adds a third-party-tool report
template to a high-traffic OSS repo"* is a recognizable funnel
shape: the value proposition reads as helpful but the long-term
effect is to redirect attention from the host project's own
abstractions toward the external tool's data model. Reviewers
should pattern-match on `## Report Template` blocks emitting
structured JSON with the originating tool's field names — that's
almost always the tell. Generic frameworks (12-layer stacks,
universal severity scales) added to project-specific repos
typically degrade docs quality even when written in good faith,
because they teach readers a vocabulary the project itself doesn't
use, creating a permanent translation tax.
