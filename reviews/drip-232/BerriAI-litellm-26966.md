# BerriAI/litellm#26966 — [Fix] Release Workflow: Detect SemVer-Style Pre-Release Dev Tags

- **Author:** yuneng-jiang (yuneng-berri)
- **Head SHA:** `cc917993a918dab1d7895ad95d38a0cee34a8d0e`
- **Base:** `litellm_internal_staging`
- **Size:** +4 / -2 across 1 file
- **Files changed:** `.github/workflows/create-release.yml`

## Summary

Closes a real GitHub-release-classification bug where `1.84.0-dev.2` (SemVer/Docker-style hyphen-dev shape) was being published as a *stable* GitHub release because the regex deciding pre-release status only matched the PEP 440 dot-dev form (`\.dev`). Per the release design doc's PyPI ↔ Docker mapping table, both shapes are valid production-track release tags for the same channel, and both should be marked as pre-releases. The fix is one-character: change `\.dev` to `[-.]dev` in the workflow's `isPrerelease` regex, accepting either separator.

## Specific code references

- `create-release.yml:49`: the load-bearing one-character regex fix. Old predicate: `const isPrerelease = /(?:rc|nightly|alpha|beta|\.dev)/i.test(tag);` New predicate: `const isPrerelease = /(?:rc|nightly|alpha|beta|[-.]dev)/i.test(tag);`. The `[-.]` character class accepts either `.` (PEP 440 `1.84.0.dev2`) or `-` (SemVer `1.84.0-dev.2`). Other prerelease markers (`rc`, `nightly`, `alpha`, `beta`) keep their existing wide-but-safe substring shape because those tokens are already unambiguous in tag context.
- `create-release.yml:48`: new two-line comment above the regex documenting the intent — `// Accept both PEP 440 (\`.dev\`) and SemVer (\`-dev\`) separators so tags // like \`1.84.0.dev2\` and \`1.84.0-dev.2\` are both detected.` This is exactly the right place for the comment because the regex is otherwise opaque to anyone not steeped in the dual-versioning convention; the next maintainer who sees a "false positive" pre-release marking will need this context to know it's intentional.
- `create-release.yml:7`: `inputs.tag.description` widened to include the new hyphen-dev form as an example: `1.84.0, 1.84.0rc1, 1.84.0.dev42, 1.84.0-dev.2, 1.84.0.post1; legacy v1.83.10-stable still accepted`. Symmetric with the regex change — anyone hand-dispatching the workflow now sees the SemVer dev shape in the inline help and is less likely to be confused about which shape to use.

## Reasoning

Three things make this exemplary: (a) PR body includes a 16-tag classification matrix (`PASS  1.84.0-dev.2  want prerelease  got prerelease`...) covering every relevant shape — new (PyPI canonical, Docker SemVer, chained pre-releases like `1.86.0rc2.dev3`) and legacy (`v1.83.X-nightly`, `vX.Y.Z-stable`, `-stable.patch.N`); (b) the offline verification approach is sound and explicitly documented because dispatching the workflow for runtime verification would create a real GitHub release object (disruptive even with throwaway tag); (c) the change is minimally scoped — one workflow file, one character of regex, one comment, one description-string update. Risk is essentially zero because the new pattern is a strict superset of the old one (every tag the old regex matched as prerelease still matches; only the new hyphen-dev shape is added).

One nit: the `[-.]dev` character class would also match an exotic `1.84.0+dev.2` if someone ever started using a `+` separator (it wouldn't because `+` isn't in `[-.]`), but it would *also* match unusual sequences like `nondev1.84.0` if such a tag existed because there's no word boundary anchor. The risk in practice is nil since release tags never contain English words like `nondev`, but a `(?:^|[-.])` anchor would be marginally safer. Not blocking — defer to the existing tag-shape conventions.

The first end-to-end runtime confirmation of this fix will be the next dev/RC release on `1.84.0-dev.N` shape; the PR's "Test plan" correctly leaves that as an open checkbox. Worth coordinating with the next release-cut owner so they can verify the GitHub release UI shows the pre-release badge as expected.

## Verdict

**merge-as-is**
