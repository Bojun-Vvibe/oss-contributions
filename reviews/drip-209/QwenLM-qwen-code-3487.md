# QwenLM/qwen-code#3487 — refactor(bun): clean up bundled mode implementation

- PR: https://github.com/QwenLM/qwen-code/pull/3487
- Head SHA: `b9adfd78e660bd03bb43e601529e4bf9091c0366`
- Author: DennisYu07
- Files: 17 changed, +1400 / −173

## Context

The qwen-code core package has been growing a Bun-bundled-mode story in parallel
with the Node entry path: `bun build --compile`-able binary, runtime detection
(`isInBundledMode()`, `isRunningWithBun()`), bundled vs. on-disk skill loading,
and feature flags injected at compile time via `--define`. Three problems had
accumulated:

1. **Duplicate detection helpers.** `bundle-features.ts` and `bundledMode.ts`
   both defined `isInBundledMode` / `isRunningWithBun` / `getRuntimeMode`,
   with subtly different fallback orders.
2. **`require()` in `yaml-parser.ts`.** A hand-rolled YAML parser (~190 LOC of
   line-walking state machine) lived in core, with a dynamic `require('yaml')`
   conditional that lint refused to bless.
3. **Shape drift in `bundledSkillManager.ts`.** Type-unsafe `any`-typed
   platform-compat shims for accessing the Bun-embedded skill bundle.

## Design

This PR is best read as a small infrastructure refactor with one substantive
correctness improvement (the YAML-parser swap) wearing the clothes of a much
larger diff (most of the volume is new files: `bundle-features.ts`,
`bundledMode.ts`, `bundledSkillManager.ts`, `crypto-browser.ts`,
`crypto-runtime.ts`, `fastMode.ts`, `yaml-parser-base.ts`, etc.).

**Feature-flag registry (`bundle-features.ts:30-67`).** New typed
`FeatureName` union of compile-time flags (`VOICE_MODE`, `TEAMMEM`, `KAIROS`,
`COORDINATOR_MODE`, `AGENT_TRIGGERS`, `TRANSCRIPT_CLASSIFIER`,
`BASH_CLASSIFIER`, `PROACTIVE`, `FAST_MODE`, `MCP`, `SKILLS`, `CORE`).

```ts
export function feature(name: FeatureName): boolean {
  const envKey = `FEATURE_${name}`;
  const envValue = process.env[envKey];
  if (envValue !== undefined) {
    return envValue === 'true' || envValue === '1';
  }
  return featureDefaults[name] ?? false;
}
```

The `process.env[envKey]` indirection is what enables Bun's `--define
process.env.FEATURE_X='true'` to compile-time-substitute a literal that
DCE can eliminate the `false` branches of. This is the right shape — the
function looks like a runtime check but optimizes to a literal in the
compiled binary. The `featureDefaults` map gives Node fallback behavior so
non-bundled runs (test, dev) still work without setting every env var.

**YAML parser swap (`yaml-parser.ts:4-115`).** The hand-rolled state-machine
parser is replaced with:

```ts
/**
 * Uses Bun.YAML (built-in, zero-cost) when running under Bun,
 * otherwise falls back to the `yaml` npm package.
 * The package is lazy-required inside the non-Bun branch
 * so native Bun builds never load the ~270KB yaml parser.
 */
```

This is the most impactful change in the PR. The pre-PR parser handled "basic
key-value pairs, arrays, and nested objects" via line indentation matching,
which is exactly the YAML subset that breaks the moment a real subagent
frontmatter ships an inline-flow object (`description: { tone: brief }`),
multi-line block scalar (`prompt: |`), or quoted string with embedded
colons. Bun's built-in `Bun.YAML` and the `yaml` npm package both handle
the full spec; the lazy-require gate inside the non-Bun branch keeps the
~270KB cost out of compiled SEA binaries. The new
`yaml-parser-base.ts` (193 LOC) holds the shared types and the
fallback-decoder logic.

**Detection unification (`bundledMode.ts:1-100`).** The canonical home for
`isInBundledMode()`, `isRunningWithBun()`, `getRuntimeMode()`. `bundle-features.ts`
now re-exports them from here rather than redefining them. This collapses the
prior dual-source-of-truth.

**Skill manager (`bundledSkillManager.ts`, 210 LOC, new).** Centralizes
loading skills from the Bun-embedded asset bundle vs. the filesystem,
behind a single `BundledSkillManager` API that the rest of core can use
without conditionals.

**Crypto runtime split (`crypto-browser.ts`, `crypto-runtime.ts`, new).**
Splits the crypto wrappers so the browser-style WebCrypto path lives in one
file and the Node `node:crypto` path lives in another, with the runtime
choice made at module-load via the unified detection from `bundledMode.ts`.

**Build config (`bun.build.config.ts`, 129 LOC, new) and
`scripts/build-native.js` (116 LOC, new).** The compile-time wiring that
applies the `--define` substitutions, points at the bundled-skill manifest,
and produces the SEA artifact.

**Utils index (`utils/index.ts`, 40 LOC, new).** Single re-export entry
point so consumers don't need to know which file inside `utils/` exports
which symbol — pairs naturally with the rename/move churn the rest of the
PR causes.

## Risk analysis

**Behavior parity for the YAML swap.** The hand-rolled parser handled a
narrow subset (basic key-value, arrays, nested objects) — but real subagent
frontmatter files in the repo are constrained to *that* subset by
convention. Switching to a full YAML parser is correct-bias-positive
(things that should parse will), but worth a regression test that asserts
existing subagent frontmatter files still produce the same parsed object.
The PR description doesn't mention such a test. Spot-checking with a
representative subagent file under the new parser would be the right move
before merge.

**Lazy-require classification under Bun.** The lazy-require pattern
(`require('yaml')` inside the non-Bun branch) needs to be invisible to
Bun's bundler. If Bun's static analysis sees the `require('yaml')` call
and resolves it eagerly, the 270KB savings disappears. The right
verification is `bun build --compile && du -sh dist/qwen` before and after
the swap; the PR description doesn't include that measurement.

**Feature-flag default for `MCP` / `SKILLS` / `CORE`.** Three flags
default to `true`. The other nine default to `false`. This means a
"pure-default" build (no `--define` overrides) ships with MCP, skills,
and core — the right minimum runnable surface. But there's no
type-system gate to stop a future PR from shipping a feature default-on
without realizing it expands the SEA binary. A `meta.test.ts` that
asserts `featureDefaults` against a frozen baseline would catch that.

**Concentration risk.** 17 files, 1400+/173- LOC, touching the build
system, runtime detection, YAML parsing, crypto, and feature flags in a
single PR is a lot. Each component change is reasonable in isolation,
but landing them together makes a partial revert (e.g. "the YAML swap
broke X, roll just that back") meaningfully harder. Splitting into:
- (a) detection unification (`bundledMode.ts` + remove dups)
- (b) YAML parser swap (`yaml-parser.ts` + base)
- (c) skill manager + crypto runtime split
- (d) build config + utils index

would each be reviewable in minutes and revertable independently.

## Verdict

**`merge-after-nits`**

The directional changes are all correct: dedup the runtime detection,
delete the home-rolled YAML parser, type the feature-flag registry, give
core a single utils entry point. Three blockers in current shape:

1. **Add a behavior-parity test for the YAML swap.** Pick the existing
   subagent frontmatter files in the repo and assert the new parser
   produces an equivalent object. The hand-rolled parser was permissive
   in subtle ways (it ignored comments after `#`, collapsed certain
   indentation patterns); the new parser is stricter. A diff in either
   direction is acceptable, but it should be intentional, not silent.
2. **Verify the Bun-DCE story for the lazy `require('yaml')`.** Include
   the `bun build --compile` size measurement before/after in the PR
   description. If the bundler eagerly resolves the require, the 270KB
   savings is illusory and the lazy-require should be a dynamic
   `await import()` inside `isRunningWithBun() === false` branches
   instead.
3. **Split the PR.** The runtime-detection unification is a 1-day
   merge; the YAML swap deserves a focused review with the parity test;
   the build-config additions deserve a separate review with someone
   who owns release. Landing all of it in one commit makes any partial
   regression a full-PR revert.

The feature-flag registry is the cleanest piece and could land
independently and immediately.

## What I learned

The compile-time-feature pattern via `process.env.FEATURE_X` is a clever
way to make runtime checks DCE-able without inventing a custom build-flag
syntax — the same source code reads as a normal env-var check during
dev/test (where `featureDefaults` is the source of truth) and gets
substituted-and-eliminated at SEA-compile time. The trap is that any
feature gated by it can't have side effects in its `import` chain that
matter at module-load time, because the import will still run before the
DCE pass kicks in on the function bodies. The lazy-require around `yaml`
is the right shape for this; making the same pattern the convention for
all flag-gated subsystems is a discipline worth documenting alongside
the new `bundle-features.ts`.
