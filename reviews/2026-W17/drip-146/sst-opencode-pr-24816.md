# sst/opencode #24816 — fix(acp): accept https:// URIs in image content blocks

- PR: https://github.com/sst/opencode/pull/24816
- Head SHA: `3cfbef4ed260626aba492ee32f1d96373732d538`
- Author: truenorth-lj
- Files touched: `packages/opencode/src/acp/agent.ts` (1 line change)

## Observations

- `packages/opencode/src/acp/agent.ts:1394` — the existing branch only matched `part.uri.startsWith("http:")`, which silently dropped any `https://` image URI delivered through the ACP `image` content block. The PR replaces the single prefix check with `(part.uri.startsWith("http://") || part.uri.startsWith("https://"))`, restoring parity with how other tools accept remote image URLs.
- The change is surgical and behavior-preserving for `http://` URIs (the original `"http:"` test also matched `https:` due to substring nature — wait: `startsWith("http:")` *would* also match `https:`, since `https:` starts with `http`? No — `"https:".startsWith("http:")` is **false** because position 4 differs (`s` vs `:`). So the prior code genuinely rejected https. Good catch.).
- No tests added; the ACP image flow doesn't have unit coverage in this file. A small regression test (`https://example.com/foo.png` → routed into `parts.push({ type: "file", url })`) would be cheap insurance.

## Verdict: `merge-after-nits`

**Rationale:** One-line correctness fix for a real bug (https image URIs were being silently dropped). Worth landing as-is, but a one-liner test would prevent regression — the rest of the `parts.push` shape is exercised by neighboring branches.
