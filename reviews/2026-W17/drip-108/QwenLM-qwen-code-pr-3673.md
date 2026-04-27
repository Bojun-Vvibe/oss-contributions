# QwenLM/qwen-code #3673 — feat(memory): add autoSkill background project skill extraction

- **Repo**: QwenLM/qwen-code
- **PR**: #3673
- **Author**: LaZzyMan
- **Head SHA**: e05e614afa04d84691137894389093397c54ecb3
- **Size**: +2154 / −12 across ~19 files (large new feature surface)

## What it changes

Adds an opt-in **autoSkill** background extraction
pipeline that periodically reviews the project's
conversation history with a planner agent and either
proposes new skills or refines existing ones, surfaced
to the user as a "skill nudge" in the TUI.

Six structural pieces:

1. **New design doc** at
   `docs/design/skill-nudge/skill-nudge.md` (+460 lines)
   describing the architecture: a memory manager
   subsystem schedules skill-review tasks, a planner
   agent (`runSkillReviewByAgent`) consumes recent
   conversations and emits a structured nudge, the TUI
   renders the nudge non-blockingly.

2. **Settings schema entry** at
   `packages/cli/src/config/settingsSchema.ts:1123-1133`
   adding the `autoSkill` field (10 lines, default
   off), wired through to `Config` at
   `packages/core/src/config/config.ts:462-464` and
   `:692-693` with a `getAutoSkillEnabled()` accessor.

3. **MemoryManager extension** at
   `packages/core/src/memory/manager.ts:114-185` adding
   a new `scheduleSkillReview` task type alongside the
   existing `extract`/`drain` paths, with
   `MemoryTaskStatus` widened to include the
   `'reviewing-skills'` state. The +172/−3 in this file
   is mostly the scheduler arm wiring.

4. **New planner module** at
   `packages/core/src/memory/skillReviewAgentPlanner.ts`
   (+240 lines, brand new file) that builds the prompt,
   dispatches to the agent runtime, parses the
   structured response, and surfaces it through a
   `SkillReviewByAgentResult` type. Consumed at
   `manager.ts:964` (via `runSkillReviewByAgent` import).

5. **New `SkillManageTool`** at
   `packages/core/src/tools/skill-manage.ts`
   (+261 lines, brand new file) registered in
   `agent-core.ts:339` so the planner agent can
   actually create / update / delete skills as part of
   the review. Backed by 209 lines of unit tests at
   `skill-manage.test.ts`.

6. **Integration test** at
   `packages/core/src/memory/skillReviewNudge.integration.test.ts`
   (+436 lines) end-to-end exercising
   manager → planner → tool → nudge.

Plus wiring through `nonInteractiveCli.ts` (3 hunks),
`useGeminiStream.ts` (4 lines), `client.ts` (+41/−5),
`agent.ts`, and `vscode-ide-companion/schemas/settings.schema.json`.

## Strengths

- **Disciplined feature gating.** The whole subsystem
  is gated on `config.getAutoSkillEnabled()` and
  defaults to off — non-opt-in users see zero behavior
  change. The gate is checked at
  `packages/core/src/core/client.ts:692-694` before
  even constructing the `SkillReviewParams`, so the
  feature has near-zero off-path overhead.
- **Tool-agent split is the right architecture.** The
  planner agent (in `skillReviewAgentPlanner.ts`)
  decides *what* to review and produces a structured
  proposal; the `SkillManageTool` is a separate
  surface the agent can invoke to actually persist
  changes. This keeps the planner's "read recent
  conversation, decide what's worth a skill" prompt
  decoupled from the persistence layer's "validate the
  payload, write to disk, dedupe by name" mechanics.
  A future shift from agent-driven to
  rule-driven (or the reverse) would change one
  module without disrupting the other.
- **Heavy test coverage for the new surface.**
  - `skill-manage.test.ts` (+209) covers the tool's
    create/update/delete arms with a tempdir-backed
    fixture (`fs.mkdtemp(path.join(os.tmpdir(), 'skill-manage-'))`),
    so test isolation is guaranteed.
  - `manager.test.ts` (+176) covers the new
    `scheduleSkillReview` arm against the existing
    extract/drain test scaffolding.
  - `skillReviewNudge.integration.test.ts` (+436)
    pins the full pipeline.
  - The skill-paths module gets +48 lines of
    coverage at `skill-paths.test.ts`.
  - `agent-core.ts:335` gets `enabled: autoSkillEnabled`
    threading so the agent runtime knows whether the
    feature is live.
- **Settings schema parity** between the CLI side
  (`settingsSchema.ts`) and the VS Code companion side
  (`vscode-ide-companion/schemas/settings.schema.json`,
  +5) — both surfaces describe the same opt-in. This
  matters because the VS Code companion is the
  surface most likely to expose `autoSkill` to a UI
  toggle, and schema drift would cause silent type
  errors there.

## Concerns / nits

- **Periodic-review cadence is not visible in this
  diff.** The design doc (which I haven't read in
  full) presumably specifies how often `scheduleSkillReview`
  fires, but the diff doesn't show a timer or a
  message-count threshold. If the trigger is "every N
  user turns" it's important that N is exposed as a
  setting (so users can tune the latency-vs-cost
  tradeoff) and that the default is conservative
  enough not to double the model spend per session.
  Worth a one-line PR-body sentence pointing at the
  trigger and threshold.
- **Cost transparency is missing.** The planner agent
  runs in the background and consumes tokens against
  the user's quota. There's no surface (e.g. in
  `/stats` or in the nudge itself) that says "this
  skill review cost X tokens / $Y" — making the
  feature attractive to users who want it but
  invisible-budget-cost to users who enable it
  without thinking. Pairs poorly with #3668's
  `billing.modelPrices` work (drip-107) which would
  be the natural home for autoSkill cost attribution.
  Either gate this on a "show cost in nudge" subflag
  or attribute autoSkill spend separately in the
  stats panel.
- **`SkillManageTool` write semantics need a deletion
  audit trail.** The diff shows it can delete
  skills (test coverage at `skill-manage.test.ts`).
  If the planner agent decides a skill is "stale" and
  deletes it, the user has no obvious recovery path
  unless skills are versioned or backed up before
  deletion. Worth either gating delete on user
  confirmation through the nudge flow, or writing
  to a `~/.qwen-code/skills/.trash/` directory
  rather than `unlink`.
- **The planner-agent prompt isn't visible in this
  review** (lives in `skillReviewAgentPlanner.ts`
  +240). If it can be jailbroken via conversation
  content (the conversation *is* the input), a
  malicious user could potentially get the planner
  to write arbitrary skill content via
  `SkillManageTool`. Worth confirming the planner
  validates structured output schema before
  dispatching to the tool, and that
  `SkillManageTool.build()` validates the payload
  against the same schema (defense in depth).
- **Telemetry sink for nudge accept/reject.** If
  users dismiss every nudge, the feature should
  stop pestering them — but there's no signal in the
  diff that the nudge result loops back into a
  "should I keep nudging?" decision. Worth either a
  `dismissCount` heuristic in the manager, or a
  user-facing "stop suggesting skills" command.
- **Two unrelated imports added** at
  `manager.ts:40` and `:65` and the `MemoryManager`
  test file mocks `./skillReviewAgentPlanner.js` at
  `:746`. Standard, but the test mock should ideally
  use `vi.hoisted()` to ensure the mock is in place
  before `manager.ts` imports it — race conditions
  with module-level imports have bitten qwen-code's
  test suite before.

## Verdict

**needs-discussion.** This is a substantial new
feature surface (+2100 lines, new tool, new manager
subsystem, new agent prompt path) that's
architecturally sound and well-tested at the unit
level, but introduces three cross-cutting concerns
that deserve maintainer alignment before merge: (1)
budget/cost attribution for the background agent
calls, (2) deletion-recovery and prompt-injection
defense for the `SkillManageTool` surface, and (3)
the trigger cadence and dismiss-count loop. Each is
a small follow-up but the combination shifts from
"nice opt-in feature" to "nice opt-in feature
without surprises" and the PR's design doc is the
right place to spell those out.

## What I learned

When a CLI feature spawns background work that
costs money (LLM tokens), the cost-attribution
surface needs to be co-designed with the feature,
not retrofitted. Otherwise the feature ships, users
enable it, the bill arrives, and the user can't
tell whether the bill is from their interactive
session or from the background subsystem. Pairing
opt-in features that consume tokens with opt-in
visibility into how much they consume is the
discipline; #3673 is well-architected on the
feature axis but hasn't done that pairing, which
is what makes it "needs-discussion" rather than
"merge-after-nits". The same pattern applies to
any other background scheduler that calls a model
(skill drain, memory consolidation, etc.) — they
all need a cost attribution sink before they ship.
