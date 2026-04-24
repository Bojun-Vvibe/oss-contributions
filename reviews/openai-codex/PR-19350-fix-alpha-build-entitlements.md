# openai/codex PR #19350 — fix alpha build (entitlements trim)

Link: https://github.com/openai/codex/pull/19350
State: MERGED · Author: jif-oai · +0 / −8 · Files: 1

## Summary
Strips four entitlement keys from `.github/actions/macos-code-sign/codex.entitlements.plist`:
`com.apple.application-identifier`, `com.apple.developer.team-identifier`, and the
`keychain-access-groups` array (all bound to a specific Apple Team ID), leaving only
`com.apple.security.cs.allow-jit`. Goal: unblock the alpha signing pipeline, presumably because
those Team-ID-bound keys collide with the alpha signing identity (different team) or the alpha
provisioning profile no longer asserts them.

## Strengths
- Minimal, surgical CI fix — touches one CI-only file, zero runtime code.
- Keeps `allow-jit`, which is the entitlement that actually matters for codex's process model.
- Removes hard-coded Team IDs from a checked-in plist, which is good hygiene regardless: those
  values were a constant tripwire if the team identity ever rotates.

## Concerns / Risks
- **Edge case: keychain access loss.** Removing `keychain-access-groups` means signed binaries can
  no longer share keychain entries with sibling apps under the same access group. If any login or
  refresh-token flow relies on items written into the team-scoped keychain group, those items
  become invisible after this build, manifesting as silent re-auth prompts on first launch of an
  alpha build. No test or CI assertion guards this regression — keychain reads degrade gracefully
  to "item not found", so a regression here would not fail the build, only end users.
- **No corresponding entry in the PR body** explaining *why* the alpha build broke. Future
  contributors hitting the same signing failure on prod / beta will not be able to learn from
  this commit. A two-line note in the plist or commit message would have prevented the
  knowledge loss.
- **No accompanying change** to the prod entitlements path. If alpha and prod share signing
  infrastructure but only one plist was touched, the next prod build may fail in the same way and
  someone will reach for the same blunt fix without checking whether dropping the keychain group
  is acceptable for prod (which has real users with persisted keychain items).
- **Reversibility**: a future Team-ID rotation or a new entitlement requirement (e.g.
  `com.apple.developer.networking.networkextension`) would require re-introducing structured
  per-environment plists. The PR removes the only on-disk record of what the alpha team ID was;
  reconstructing it later requires git archaeology.

## Suggested follow-ups
1. Document, in `.github/actions/macos-code-sign/README.md` or the action's `action.yml`, which
   entitlements are *intentionally* injected by the signing identity vs. checked in.
2. Add a smoke test that launches the signed alpha binary and exercises one keychain read/write
   round-trip, so a future "drop the keychain group" change cannot silently break refresh-token
   persistence.
3. Audit `prod` and `beta` entitlements plists for the same Team-ID hardcoding and decide
   whether to template them via the signing action rather than commit them.
