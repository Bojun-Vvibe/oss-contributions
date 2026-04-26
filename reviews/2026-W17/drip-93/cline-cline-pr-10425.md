---
pr: 10425
repo: cline/cline
sha: 443cd38da227b8c8a52f92c7bf2482414d9c2ee2
verdict: merge-after-nits
date: 2026-04-27
---

# cline/cline #10425 — [Aikido] Fix 26 security issues in node-forge, @xmldom/xmldom, basic-ftp and 7 more

- **Author**: aikido-autofix (bot)
- **Head SHA**: 443cd38da227b8c8a52f92c7bf2482414d9c2ee2
- **Link**: https://github.com/cline/cline/pull/10425
- **Size**: +129/-157 in `package-lock.json` (and root `package.json` for the direct `axios` bump).

## Scope

Bot-generated security upgrade resolving 26 CVEs across nine packages. Direct deps: `axios 1.15.0 → 1.15.1` (root). Transitive bumps via lockfile: `node-forge 1.3.3 → 1.4.0` (CRITICAL CVE-2026-33896 — `verifyCertificateChain` accepts leaves-as-CAs when intermediates lack `basicConstraints`; HIGH CVE-2026-33891 modInverse infinite loop; HIGH CVE-2026-33894 PKCS#1 signature forgery via Bleichenbacher; HIGH CVE-2026-33895 Ed25519 non-canonical signature acceptance), `@xmldom/xmldom 0.8.11 → 0.8.13` (4 HIGH XXE/injection CVEs), `basic-ftp 5.2.0 → 5.3.0` (HIGH CRLF injection + GHSA), `path-to-regexp 8.3.0 → 8.4.2` (HIGH ReDoS), `hono 4.12.9 → 4.12.15` (HIGH path traversal in `toSSG()`), `fast-xml-parser 5.5.8 → 5.7.1`, `follow-redirects` (header leak across cross-domain redirects), `@hono/node-server 1.19.11 → 1.19.14`, `axios → 1.15.1` (prototype pollution).

## Specific findings

- `package.json` direct dep change at the root: `axios 1.15.0 → 1.15.1`. Visible in `package-lock.json:58` block (`"axios": "1.15.1"`). Only direct-dep change; everything else is transitive.
- `package-lock.json:1344-1351` — `@aws-sdk/xml-builder 3.972.15 → 3.972.19` brings `fast-xml-parser` from 5.5.8 → 5.7.1 (resolves CVE-2026-41650, XMLBuilder failing to escape `-->` in comments and `]]>` in CDATA). The bot's analysis correctly identifies that this is only used by AWS SDK to parse trusted AWS responses, so the security restrictions don't affect normal operations — but they *do* harden against a hypothetical compromised-AWS-response scenario.
- `package-lock.json:2941-2946` — `@hono/node-server 1.19.11 → 1.19.14` (CVE-2026-39406: `serveStatic` middleware bypass via repeated slashes).
- The bot's breaking-change analysis (incomplete: 6/10 analyzed) covers the four that *did* get analyzed. Importantly:
  - **node-forge 1.3.3 → 1.4.0** — body claims it's "transitive through `ssh2` but the codebase does not import ssh2 / node-forge / verifyCertificateChain etc." That's a falsifiable claim; a maintainer should grep the source tree to confirm there's no direct `forge.pki.*` usage anywhere.
  - **@xmldom/xmldom 0.8.11 → 0.8.13** — used transitively through `js-yaml` and `pdf-parse`. Bot says codebase only does read/parse and never `createCDATASection`. Plausible.
  - **basic-ftp** — claimed transitively reached via `data-uri-to-buffer`. The codebase doesn't do FTP work, so the new directory-listing upper bound shouldn't bite.
  - **path-to-regexp 8.3.0 → 8.4.2** — bot claims "doesn't define routes with large numbers of optional groups", which is correct for normal Express-style routes but worth double-checking against the actual layer-routing code in cline.
- Breaking-change analysis is **NOT** available for: `hono`, `follow-redirects`, `@aws-sdk/xml-builder`, `@hono/node-server`. This is the meaningful unknown — a maintainer needs to scan the changelogs for those four manually before merging. `hono 4.12.9 → 4.12.15` is six minor patch versions in one go; non-trivial.
- `package-lock.json:1581-1582,2879-2882,3922-3923` — three new `"peer": true` annotations on `@babel/core`, `@grpc/grpc-js`, `@opentelemetry/api`. Lockfile-shape change, not a version change. Probably benign (npm normalizing peer-dependency markers), but unexpected in what's billed as a security PR.
- `package-lock.json:5614-5618` — `@rollup/rollup-android-arm` loses its `peer: true` marker. Same shape-change category.
- `package-lock.json:3868-3880` — net new package `@nodable/entities@2.1.0` shows up in the lockfile. New transitive supply-chain entry pulled in by one of the bumps. Worth a once-over to confirm provenance.

## Risks

- **CRITICAL CVE-2026-33896** in node-forge (cert-chain validation bypass) is the headline reason to merge — even if the bot is right that the codebase doesn't directly use `verifyCertificateChain`, transitive dependents inside `ssh2` might. Sitting on this is worse than the small risk of a transitive breaking change.
- The bot's "no breaking-change analysis" gap on `hono` / `follow-redirects` / `@hono/node-server` / `@aws-sdk/xml-builder` is the main merge-blocker concern. None of these are likely to bite, but bot self-attestation isn't a substitute for a maintainer eyeballing the four uncovered changelogs.
- Net-new transitive `@nodable/entities` package warrants a quick provenance check (not flagged by the bot).

## Verdict

**merge-after-nits** — the security payoff is large (26 CVEs, including one critical), the lockfile churn is mostly mechanical, and the direct-dep surface is one tiny axios patch bump. But the four packages the bot couldn't analyze for breaking changes (`hono`, `follow-redirects`, `@hono/node-server`, `@aws-sdk/xml-builder`) plus the net-new `@nodable/entities` transitive dep need a five-minute manual changelog/provenance check before this lands. Don't auto-merge; do merge after that check.
