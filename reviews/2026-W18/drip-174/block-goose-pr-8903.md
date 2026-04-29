# block/goose #8903 — chore: add UI refactor review skill for goose2

- **PR:** https://github.com/block/goose/pull/8903
- **Title:** `chore: Added goose 2 UI refactor review skill`
- **Author:** lifeizhou-ap (Lifei Zhou)
- **Head SHA:** 3eab3c261d8faa2e77d0562edc975477ea208bf5
- **Files changed:** 1 new (`.agents/skills/ui-refactor-review/SKILL.md`), +247 / 0
- **Verdict:** `merge-after-nits`

## What it does

Adds a single new agent skill file at
`.agents/skills/ui-refactor-review/SKILL.md` (+247) intended for
ui/goose2 contributors. The skill defines a structured review workflow
focused on maintainability/refactor quality (not correctness) for React
+ TypeScript + Tauri code under `ui/goose2`. Sections cover: Goals,
Workflow (10 steps), Strict Mode, Smell Checklist (7 questions), and
Rules covering Size/Decomposition, Naming, Layer Discipline, Module
Encapsulation, DRY/Hooks, Type Hygiene, React/UI, Notifications/i18n/a11y,
Tauri/Backend Boundaries, Errors/State Drift/Dead Code (truncated at
the diff window I sampled).

The skill is markdown-only, no code, no tests, no CI integration. It's
a contribution to the agent skill library at `.agents/skills/`, not to
shipped product code.

## Why it's correct

- Strictly additive: no existing files modified, no behavior change,
  no risk to runtime code paths.
- Lives in the right directory (`.agents/skills/`) which is the
  established convention for agent-facing skill definitions.
- The frontmatter `name`/`description` follows the standard skill
  format other skills in the repo presumably use.
- Author tested it against an actual PR (`#8896`) per PR body — at
  least a smoke-test of the workflow happened.

## What's good

- The skill is opinionated in the right ways:
  - "Review the post-PR shape. Do not grade on a curve" (Strict Mode)
    — explicitly counters the common reviewer failure mode of
    grading partial cleanup as "good enough."
  - The Smell Checklist forces an iteration step before the reviewer
    is allowed to finalize, which is a real guard against
    "I-found-three-things-let's-stop" early termination.
  - Layer Discipline section is concrete: explicit lists of what
    belongs in `ui/`, `hooks/`, `api/`, `lib/`, `stores/` instead of
    generic "separation of concerns" hand-waving.
  - The "Tauri And Backend Boundaries" section is specifically:
    "Frontend-to-core communication goes through `SDK -> ACP -> goose`.
    Do not add ad hoc `fetch()` calls for goose core behavior. Do not
    add `invoke()` calls as proxies to goose core behavior; reserve them
    for desktop-shell concerns." That's the kind of specific
    architectural rule that prevents real PRs from drifting.
- Smell thresholds are stated as smells, not hard limits ("around 200
  lines", "around 40 lines") — correctly leaves room for justified
  exceptions.
- `Module Encapsulation` rule "If a helper is used in only one module,
  default to keeping it local. If similar helpers appear across two
  modules, default to extracting them" gives a concrete extraction
  threshold (2 sites) instead of the vague "rule of three."

## Nits / risks

1. **The skill is huge for what it does.** 247 lines of prose in a
   single SKILL.md is at the upper end of "an agent will read every
   line carefully each invocation." Skills loaded into a context
   window cost tokens; a 247-line skill that gets pulled on every
   ui/goose2 PR review is a real recurring cost. Consider:
   - Moving the Rules section into a separate `RULES.md` referenced
     from `SKILL.md`, so the agent only loads it on demand.
   - Trimming the Workflow section from 10 numbered steps to 5-6
     by collapsing 7 (run a second pass) and 8 (output order) into
     "After enumerating issues, run one discovery-only second pass,
     then emit Applied Well / Issues / Checklist in that order."
2. **Frontmatter `description` is well-formed but lacks negative
   triggers.** Other skills in the broader skill ecosystem typically
   include "Does NOT handle: …" so the skill router doesn't pick it
   for adjacent tasks (e.g. backend Rust review, ui/goose1 cleanup).
   Add a "Does NOT handle: ui/goose1 changes (different
   architecture); backend Rust review (use rust-review skill);
   feature-correctness review (use code-review skill)" clause.
3. **No example invocation.** The PR body lists one prompt
   ("`Review my changes (or pr link) for refactoring opportunities`")
   but the skill itself doesn't include a "Trigger phrases" list or
   "Example outputs" section. A 4-line "Example output shape"
   appendix showing what `Applied Well` / `Issues` / `Checklist`
   actually look like would help the agent calibrate.
4. **Reviewer tone is asserted, not enforced.** "Do not stop after
   the first few findings" is the right rule but agents tend to stop
   anyway when context fills up. Worth adding a hard rule like "Emit
   at least one Issue per Smell Checklist bucket that triggered, even
   if the finding is small" — gives the agent an explicit termination
   condition that resists premature stopping.
5. **Author convention:** four out of four ui-refactor-review-style
   community skills the team has been adding are by the same author
   (per the skill listings in the broader ecosystem). Worth a "this
   is one of several maintainability skills; see also …" cross-link
   so users discover the related ones.
6. **No CI / lint integration.** The PR body says "Will recommend the
   UX designer to try first and if it works well, it can be also
   referred in the normal code review skill." That's the right rollout
   strategy — but worth filing a follow-up issue for "promote to
   default code-review skill if X PRs use it successfully" so the
   trial doesn't get forgotten.
7. The "Errors, State Drift, And Dead Code" section header appears at
   the bottom of the diff window I read but the body is truncated — I
   can't confirm it's complete. Worth a final read-through to make
   sure no rule section ends mid-sentence.

## Verdict

`merge-after-nits` — useful, opinionated, well-structured skill that
solves a real problem (consistent maintainability bar on ui/goose2
PRs). Trim aggressively, add negative-trigger frontmatter, and link
related skills before merging — the content is good but token cost on
every invocation is the silent failure mode for skills like this.
