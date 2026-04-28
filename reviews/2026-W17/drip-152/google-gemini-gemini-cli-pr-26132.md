# google-gemini/gemini-cli#26132 — fix(cli): prevent automatic updates from switching to less stable channels

- **Repo:** [google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli)
- **PR:** [#26132](https://github.com/google-gemini/gemini-cli/pull/26132)
- **Head SHA:** `7ae7242fee9077caaa8a88be0b2ab4f551cc2dc0`
- **Size:** +173 / -12 across 5 files (`updateCheck.ts`+test, `handleAutoUpdate.ts`+test, `core/utils/channel.ts`)
- **State:** MERGED (fixes #24810)

## Context

Stable-channel users were being auto-updated to nightly builds. Root
cause: when nightlies are published with semver-greater versions
(e.g. `1.1.0-nightly.1` vs current stable `1.0.0`), and the npm
`latest` dist-tag occasionally pointed at the nightly version, the
update checker did `semver.gt(latestUpdate, currentVersion)` → true
and offered the nightly as an update to a stable user. The CLI had no
notion of "stability ranking" — it treated `latest` as authoritative
even when `latest` named a less-stable build.

## Design analysis

Three layers of fix, with an explicit "defense-in-depth" cadence:

1. **Pure stability-classifier** at `core/utils/channel.ts:23-31`
   exports `getChannelFromVersion(version)` returning a `ReleaseChannel`
   enum (`NIGHTLY`/`PREVIEW`/`STABLE`) and a `RELEASE_CHANNEL_STABILITY`
   numeric ordering (`{nightly: 0, preview: 1, stable: 2}`). Right
   shape — pure function with no side effects, the numeric ordering
   makes the comparison `RELEASE_CHANNEL_STABILITY[target] <
   RELEASE_CHANNEL_STABILITY[current]` a one-line monotonicity check.
   Naming the constant in caps + records it as load-bearing semantic
   rather than implementation detail.

2. **Update-check stability gate** at `updateCheck.ts:99-110` is the
   primary fix. After fetching `latestUpdate` and computing
   `targetChannel = getChannelFromVersion(latestUpdate)`, the check:

   ```ts
   if (
     RELEASE_CHANNEL_STABILITY[targetChannel] <
     RELEASE_CHANNEL_STABILITY[currentChannel]
   ) {
     return null;
   }
   ```

   Returns `null` (no update offered) before the existing `semver.gt`
   check at `:113`. The ordering is critical: stability-gate first,
   then version-gate. Without the stability-first ordering, the
   semver check would still happen and `null`-return would be reached
   only after the unnecessary work — minor perf, but the readability
   win is real.

3. **Defense-in-depth at install time** at `handleAutoUpdate.ts`
   adds a *second* identical stability check before the global install
   command spawns. This is the right shape for a security-relevant
   gate: the update-check gate is the primary "don't even tell the
   user" defense, the install-time gate is the "even if the prior
   check was bypassed by a config injection or a bug, don't actually
   execute the cross-channel install" backstop. Two checks at two
   layers, both reading from the same pure classifier, is the
   textbook defense-in-depth shape.

## Test coverage

The `updateCheck.test.ts:172-238` matrix is the strongest part:

- `should NOT offer nightly update to a stable user even if tagged as
  latest` (`:172-180`) — the exact bug-shape from #24810
- `should NOT offer preview update to a stable user even if tagged as
  latest` (`:182-191`) — the symmetric preview→stable downgrade case
- `should offer stable update to a stable user` (`:193-200`) — the
  positive baseline
- `should offer stable update to a nightly user` (`:202-216`) — the
  *upgrade* path: nightly users should be allowed to graduate to
  stable. The mock implementation at `:208-213` is the load-bearing
  detail — when the nightly-channel `latestVersion` returns the
  current nightly (no nightly update available) but the
  default-channel `latestVersion` returns a stable version, the
  user should see the stable upgrade. This pins the
  "nightly-to-stable graduation works" semantic against future
  regressions.
- `should offer stable update to a preview user` (`:218-227`) — the
  preview→stable graduation path

The `handleAutoUpdate.test.ts:362-380` defense-in-depth test
(`should NOT update if target is less stable than current
(defense-in-depth)`) constructs the scenario where the update info
was already produced (so the primary gate didn't catch it — simulating
a stale cache or upstream injection) and asserts the install-time
gate still refuses.

The 4-channel × 5-cell matrix gives roughly the right coverage:
(stable, preview, nightly) × (offer-equal, offer-higher, refuse-lower,
refuse-equal-different-stability) = ~20 cells, the PR covers the
asymmetric high-value ones.

## Risks / nits

1. **`isNightly` branch at `:74` short-circuits past the new
   stability gate.** The existing nightly-channel update path at
   `:75-93` uses `latestVersion(name, { version: 'nightly' })` and
   returns its result without going through the stability-gate
   logic. That's correct (nightly users *want* nightly updates) but
   means the stability gate only fires on the non-nightly branch.
   If a future version-policy change wants nightly users to *not*
   receive nightly updates under some conditions (e.g., a staged
   rollout), the gate isn't there to enforce it. Worth a comment
   at `:74` naming the asymmetry.

2. **`getChannelFromVersion` mock at `updateCheck.test.ts:23-30`
   duplicates the production logic.** The test mock re-implements
   the classifier (`if (...includes('nightly')) return 'nightly';
   if (...includes('preview')) return 'preview'; return 'stable'`)
   instead of calling through to the real function. If the
   production classifier ever changes its rules (e.g., adds a
   `rc` channel, changes the substring matching to suffix-only),
   the test mock will silently diverge and the tests will pass
   against a stale shadow implementation. Better: import the real
   function in the test and mock only the IO.

3. **String-substring matching in `getChannelFromVersion` is
   semver-name-fragile.** The PR-body mock shows `version.includes('nightly')`
   for the channel detection. Any version string with the substring
   `nightly` anywhere — including a hypothetical hash like
   `abc-nightly-fix-789` — would be classified as nightly. Worth
   using `semver.prerelease()` and inspecting the returned tags
   strictly.

4. **`latestUpdate` short-circuit at `:99-101` returns `null`
   silently.** The new `if (!latestUpdate) return null;` early-exit
   collapses any registry-query failure into "no update available"
   without distinguishing transient from permanent failure. Worth
   logging at `debugLogger` so support has signal when users say
   "the CLI never offers updates."

5. **No test for the *boundary* where current and target are the
   same channel but target is semver-equal.** The matrix covers
   `target > current` (offer) and `target.stability < current.stability`
   (refuse), but not `target == current` exactly. Probably handled
   by the existing `semver.gt` check at `:113` returning false, but
   not asserted.

## Verdict

**merge-as-is.** The fix is correctly minimal, well-layered (pure
classifier + two gate sites), and the test matrix covers the
asymmetric high-value cells. Defense-in-depth at install time is
the right pattern for any security-relevant gate. Nits are
post-merge improvements; the substring-matching nit (#3) is the
one most worth tracking as a follow-up issue.

## What I learned

- Channel-stability ordering is best expressed as a numeric ordinal
  (`{nightly: 0, preview: 1, stable: 2}`) rather than enum-comparison
  logic — the comparison `STABILITY[target] < STABILITY[current]`
  is one expression, monotonic, and trivially extensible if a new
  channel is added between two existing ones.
- "Defense-in-depth" for software-update gates means two checks at
  two phases: the *advertise* check (don't tell the user) and the
  *execute* check (don't actually install). Both reading from the
  same pure classifier keeps them in sync without coupling them to
  shared mutable state.
- Substring-based channel detection (`version.includes('nightly')`)
  is fragile against version strings that happen to embed channel
  names. Semver pre-release tag inspection is the durable signal —
  `semver.prerelease(version)` returns the parsed tags as an array,
  and the channel is derivable from the first tag deterministically.
- The "graduate from nightly/preview to stable" upgrade path is
  asymmetric to the "downgrade from stable to nightly" refused
  path. Test matrices that only cover the "refuse downgrade"
  direction miss the equally-important "allow graduate-up"
  semantic that users actually want.
