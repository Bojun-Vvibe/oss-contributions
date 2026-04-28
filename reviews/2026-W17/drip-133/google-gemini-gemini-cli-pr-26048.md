# google-gemini/gemini-cli #26048 — feat(prototype): Behavioral Tip Personalization

**Verdict:** `needs-discussion`

- Repo: `google-gemini/gemini-cli`
- PR: https://github.com/google-gemini/gemini-cli/pull/26048
- Head SHA: `4c08d4b2`
- Related: #26047
- Diff: +513 / -156 across 6 files in `packages/cli/src/ui/`
- **Author explicitly labels this a prototype: "NOT intended for merging."**

## What it does

Prototype implementation of "Behavioral Tip Personalization" derived from
the author's GSoC proposal. Three moving parts:

1. **`tips.ts`** — refactors `INFORMATIVE_TIPS` from a flat `string[]` to a
   `Tip[]` interface (`{ text, relatedFeature?, weight? }`), tagging each of
   ~150 tips with a `relatedFeature` identifier (`'settings'`, `'mcp'`,
   `'todos'`, etc.).
2. **`persistentState.ts`** — adds `featureUsageCounts` to the persisted
   state blob, incremented at instrumented call sites.
3. **`usePhraseCycler.ts`** — implements weighted selection that
   downweights tips for already-used features and upweights tips for
   unused features. Instrumentation hooks added to
   `slashCommandProcessor.ts`, `atCommandProcessor.ts`, and
   `useExecutionLifecycle.ts`.

## Design analysis

The change shape is correct as a prototype but has multiple architecture
concerns that block merging in this form. Going in order:

### Tip data structure (`tips.ts`)

The `Tip` interface is reasonable:

```ts
export interface Tip {
  text: string;
  relatedFeature?: string;
  weight?: number;
}
```

But `relatedFeature` is a free-form string with no enum constraint. The
diff hardcodes ~10 distinct values (`'settings'`, `'mcp'`, `'shell'`,
`'memory'`, `'theme'`, `'todos'`, etc.) inline across ~150 tip records.
A typo in any one of them (e.g. `'mcps'` vs `'mcp'`) will silently degrade
the personalization without any compile-time signal. The instrumentation
sites in `slashCommandProcessor.ts` reference these strings independently,
so a drift between "what the cycler counts" and "what the tips reference"
is a near-certain future bug. Should be `type FeatureKey = 'settings' |
'mcp' | ...` or a const map.

### Persistent state (`persistentState.ts`)

Storing `featureUsageCounts` on disk needs a versioning story that the
diff doesn't show. If the schema rolls forward and existing users have
state files with the old shape, the load path needs a migration or a
defensive default. PRs that touch persistent state without a migration
plan are a recurring source of "after upgrade my CLI crashed on launch"
issues. Worth confirming the load path tolerates the absent key.

### Weighted selection (`usePhraseCycler.ts`)

The conceptual model — "show users tips for features they haven't
discovered" — is sound. But the diff doesn't show the weighting math, and
the prototype risks two failure modes:

1. **Cold start:** brand-new users have all-zero `featureUsageCounts`, so
   *every* tip is "unused-feature." The selection should fall back to
   uniform random or a curated onboarding sequence; otherwise the first
   session is indistinguishable from the legacy behavior.
2. **Power user inversion:** users who heavily use one feature
   (e.g. `/mcp` 200×) get `featureUsageCounts.mcp = 200`, which under any
   linear-downweight scheme makes mcp tips effectively unreachable. After
   a year of use, a power user sees a degenerate tip distribution
   featuring only the features they don't use — which can include
   features they *intentionally* don't want (e.g. `experimental.subagents`).
   "Unused" doesn't imply "should be onboarded."

### Instrumentation (`slashCommandProcessor.ts`, `atCommandProcessor.ts`, `useExecutionLifecycle.ts`)

The instrumentation taps three call paths but the diff doesn't expose how
the feature-key is derived per slash command. If `/settings` and
`/settings list` both increment `'settings'`, that's reasonable. If a
slash-command typo (`/setings`) is silently logged as `'unknown'` or
crashes the increment, the personalization data gets noisy.

### Test coverage

The diff lists no test files. A behavioral-personalization feature with
persisted state, three instrumentation sites, and a weighted-selection
algorithm should have at least:

- Snapshot test: cold-start picks tips uniformly
- Snapshot test: after N feature uses, tips for that feature appear less
- Migration test: legacy state file (no `featureUsageCounts`) loads
  without error

## Risks

1. **GSoC-prototype-PR-as-design-document.** The PR body says explicitly
   this is not intended for merging. Treating it as a design RFC is the
   right move; the review here is "ship as RFC, not as code."
2. **Persistent-state schema change with no migration.** Highest blast
   radius if this lands as-is.
3. **Free-form `relatedFeature` strings.** Will drift between
   instrumentation sites and tip declarations.
4. **No test coverage.**
5. **No telemetry on whether the personalization actually helps.** The
   feature is justified by "users discover unused features faster," but
   nothing in the diff lets the team measure that. Should ship with a
   counter that records "tip shown → feature subsequently used within N
   sessions" so the team can decide whether the algorithm beats uniform
   random.

## Suggestions

- Convert this PR into a design doc / GitHub Discussion (or RFC issue
  under the parent #26047) with a smaller follow-up PR doing only the
  data model + migration.
- Constrain `relatedFeature` to a typed enum.
- Define the cold-start and power-user behavior explicitly before
  implementing the selection algorithm.
- Add a measurement hook so the team can A/B the personalization later.
- If the project does want a personalization prototype merged behind a
  flag, gate the entire feature behind `experimental.tipsPersonalization`
  off by default and ship only the data-model-with-migration in this PR.

## What I learned

ML-flavored personalization features have a tendency to ship as "the
algorithm" without the surrounding contract: cold-start policy, drift
detection, measurement hook, and migration story. The right way to land
one of these is to break the PR into (1) data model + migration, (2)
instrumentation behind a flag, (3) selection algorithm with stated
cold-start / power-user behavior, (4) measurement. Prototypes that bundle
all four steps into a single 500-line PR are a useful design artifact but
not a mergeable change. The author has been admirably clear about that —
the right reviewer response is to escalate to a design conversation, not
to nit the prototype.
