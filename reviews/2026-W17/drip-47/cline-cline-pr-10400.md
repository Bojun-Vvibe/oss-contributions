# cline/cline #10400 — [Aikido] Fix 26 security issues in node-forge, @xmldom/xmldom, basic-ftp and 7 more

- **PR:** https://github.com/cline/cline/pull/10400
- **Head SHA:** `0dde4acee8d5aad081bad728fb228f09005bd5a0`
- **Files changed:** 2 — `package.json` (+3/−2), `package-lock.json` (+108/−194)

## Summary

Bot-authored (Aikido) bulk dependency bump targeting CVEs across `node-forge` (cert-chain validation bypass + signature forgery — high severity), `@xmldom/xmldom` (XML injection), `basic-ftp`, and ~7 other transitive deps. The only direct manifest change is `axios` 1.15.0 → 1.15.1; everything else is transitive (resolved through `npm-force-resolutions` style deduplication in the lockfile).

## Line-level call-outs

- `package.json:8` — `axios: 1.15.0` → `1.15.1`. The release notes for 1.15.1 are minor (a single bug-fix release per the npm registry); this is safe. But this is the *only* direct dep bumped — all the headline CVEs (`node-forge`, `@xmldom/xmldom`, `basic-ftp`) are addressed indirectly through lockfile deduplication. That's brittle: a future `npm install` without the lockfile (e.g., a fresh CI cache, a `package.json`-only consumer, or a downstream that pulls cline as a dep and uses `npm install --legacy-peer-deps`) can resolve back to the vulnerable transitive versions. **For node-forge specifically — given the severity (cert chain bypass) — pin it as a direct dep with `overrides` in `package.json`** so a missing lockfile still gets the patched version.
- `package-lock.json:1344-1350` — `@aws-sdk/xml-builder` 3.972.15 → 3.972.19 silently bumps `fast-xml-parser` 5.5.8 → 5.7.1 and `@smithy/types` ^4.13.1 → ^4.14.1. `fast-xml-parser` 5.6/5.7 had behaviour changes around prototype-pollution defaults; not strictly a CVE but a behaviour-changing bump that lands without mention in the PR description. Worth a release-notes link.
- `package-lock.json:1581` and `:2882` — `"peer": true` added to `@babel/code-frame` and `@grpc/grpc-js`. This is a lockfile-format change indicating npm is now treating these as peer deps where they were previously direct/transitive. Could legitimately be a side effect of the upgrade chain; could also indicate one of the bumped packages restructured its peerDependencies and consumers now need to install these explicitly. Verify the build still resolves cleanly on a fresh `npm ci` in CI.
- The PR description includes the line "⚠️ Breaking changes analysis not available for: follow-redirects, @aws-sdk/xml-builder, @hono/node-server, brace-expansion" — that's 4 of the bumped packages where the bot **explicitly disclaims it could not verify backward compatibility**. `follow-redirects` in particular has had silent behaviour changes between minor versions affecting auth-header forwarding; `brace-expansion` is dependency hell for glob behaviour. A human needs to spot-check the diff lines for each of those four before merging.
- No tests added, no manual verification noted in the PR body. For a security-only dep bump, "the existing test suite passes" is the minimum bar — but with bot-disclaimed breaking-change uncertainty on 4 packages, "the integration tests + at least one cert-validation smoke test" is the appropriate bar. Especially for `node-forge` which is the headline fix: add a test that exercises a TLS code path using node-forge (or whichever module touches cert validation downstream).

## Verdict

**merge-after-nits**

## Rationale

Security dep bumps are net-good and the CVEs cited are real. Two concerns before merge: (1) the headline fixes are lockfile-resolution only — pin `node-forge` (and ideally `@xmldom/xmldom`) as `overrides` in `package.json` so a fresh resolve can't downgrade them; (2) the bot disclaimed breaking-change analysis on 4 of the bumped packages — a maintainer needs to spot-check those 4 diffs and run the integration suite on a fresh `npm ci`. The `axios` bump is fine. Don't merge bot-authored security bumps without a CI green and an `overrides:` block, otherwise you're one cache miss away from re-introducing the CVE.
