---
pr: 24420
repo: sst/opencode
sha: 7bbe157f1acfa233d3ba955803e589e6844f1a2b
verdict: merge-as-is
date: 2026-04-26
---

# sst/opencode #24420 — fix: correct typo in comment

- **Author**: ariane-emory
- **Head SHA**: 7bbe157f1acfa233d3ba955803e589e6844f1a2b
- **Link**: https://github.com/sst/opencode/pull/24420
- **Size**: 2-line comment fix in `nix/opencode.nix`.

## Scope

Single-character typo fix in a Nix build comment.

## Specific findings

- `nix/opencode.nix:67` — comment changed from `# bun runs sysctl to detect if dunning on rosetta2` to `# bun runs sysctl to detect if running on rosetta2`. Pure prose. The conditional `lib.optional stdenvNoCC.hostPlatform.isDarwin sysctl` on the next line is unchanged.

## Risk

Zero. Comment-only, no behavioral surface. Even the Nix evaluation is byte-for-byte identical after parsing.

## Verdict

**merge-as-is** — drive-by typo fix, exactly the kind of contribution that should land without ceremony.
