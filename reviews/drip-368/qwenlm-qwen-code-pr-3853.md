# QwenLM/qwen-code PR #3853 — feat(installer): add hosted install release alias

- Head SHA: `a205e6ccdc0a7736b18ef92022360afce061f2fa`
- Author: `yiliang114`
- Size: +33 / -1
- Verdict: **merge-after-nits**

## Summary

Adds a third release-asset entry `install` (extensionless) that is a
byte-for-byte copy of `install-qwen.sh`, intended to back a future
hosted endpoint at `https://qwen-code.ai/install` so users can run
`curl -fsSL https://qwen-code.ai/install | bash` (the
get-it-from-the-internet idiom that homebrew, rustup, deno, bun,
oh-my-zsh, etc. ship). Plumbing covers all the right surfaces:
the asset config, the release workflow's `gh release create` arg
list, the installation guide doc, the SHA256SUMS cross-reference,
and a regression test that asserts byte-equivalence against
`install-qwen.sh`.

## What the diff actually does

1. `scripts/release-asset-config.js:16-21` — new
   `INSTALLATION_ASSETS` entry: `sourcePath:
   ['scripts','installation','install-qwen-with-source.sh']`,
   `output: 'install'`, `mode: 0o755`. Pulls from the same source
   script as `install-qwen.sh`, with a comment noting they must
   stay byte-for-byte identical.
2. `.github/workflows/release.yml:425` — adds
   `dist/standalone/install` to the `gh release create` argument
   list immediately after `qwen-code-*` archives and before the
   existing `install-qwen.sh`. The line ordering puts it in the
   "installer entrypoints" bucket which is correct.
3. `scripts/installation/INSTALLATION_GUIDE.md:37,44-58` — adds
   `install` to the published-assets list, documents the alias
   relationship ("identical to `install-qwen.sh` and can be used
   interchangeably on Unix systems"), notes the hosted endpoint
   isn't live yet, and gives a release-tag-pinned `curl …| bash`
   example as the interim until the hosted alias is live.
4. `scripts/tests/install-script.test.js:277,328-419` — six new
   assertions across three tests:
   - `release-asset-config` test: asserts the new
     `output: 'install'` entry appears in the config.
   - `INSTALLATION_ASSET_NAMES` test: extends the expected array
     to `['install-qwen.sh', 'install', 'install-qwen.bat']`,
     and adds positive cases for `isInstallationAssetName('install')`
     and `isReleaseChecksumAsset('install')`.
   - `buildInstallationAssets` golden test: builds into a
     temp dir, asserts the `install` file exists, asserts
     `readScript(install) === readScript(installSh)` (the
     byte-equivalence invariant), asserts the SHA256SUMS file
     contains a line ending in `\sinstall\n` (i.e. the alias is
     hashed alongside the rest), and asserts `lstatSync(install).mode
     & 0o111 !== 0` (executable bit set on Unix).
   - Workflow-shape test: asserts the release workflow contains
     `'dist/standalone/install \\'` (with the trailing
     backslash-space-newline, pinning the line position).
   - Guide-shape test: asserts the new
     `https://qwen-code.ai/install` URL appears in the doc.

## Why merge-after-nits

The mechanism is solid and the test coverage is genuinely good
(byte-equivalence + executable-bit + checksum coverage are exactly
the right invariants for an installer alias). A few real concerns
keep this from `merge-as-is`:

1. **`isReleaseChecksumAsset('install')` is the right answer but the
   wrong test signal.** The test at line 333 asserts the new asset
   is checksummed, but doesn't assert the symmetric property: that
   *both* `install` and `install-qwen.sh` produce the *same*
   SHA256 line in the file (since they're byte-identical, they
   must). That assertion would catch a future drift where someone
   accidentally edits one source script and not the other. Cheap
   to add: `expect(sha256Line('install')).toBe(sha256Line('install-qwen.sh'))`.

2. **The `\sinstall\n` regex assertion is fragile.** The
   `expect(checksums).toMatch(/\sinstall\n/)` at line 416 will
   match `…hash  install\n` (correct) but also `…hash
   reinstall\n` (incorrect — substring collision with
   `re-install` or any other `*install` filename). Anchor it
   either as `/\s install$/m` (whitespace + literal name + line
   end) or use a startswith/endswith check on a parsed line.
   Low-probability hit today, but the file format is
   `<hash><spaces><filename>` and there's no reason not to write
   the assertion exactly that shape.

3. **`https://qwen-code.ai/install` is not yet live.** The PR
   ships the documentation referencing a domain that the project
   doesn't yet operate (per the doc's own "That hosted endpoint
   is not available yet" caveat at lines 51-54). This is fine as
   "land the asset now, configure DNS later", but it would be
   strictly safer to either (a) gate the documentation update
   behind a follow-up PR that lands when the redirect is live, or
   (b) add a TODO in the doc tagged with a tracking issue so the
   "until it is live, use a concrete release version" workaround
   gets removed once the hosted endpoint exists. As written, the
   `curl … | bash` example at line 56-57 will outlive its
   purpose silently.

4. **No mention of the hosted endpoint's TLS / signature contract.**
   `curl | bash` security is famously dependent on TLS pinning,
   redirect policy, and ideally GPG/cosign signature verification.
   The doc additions don't say anything about how the hosted
   endpoint will compare to direct GitHub release URLs in terms
   of trust (GitHub releases at least have the implicit GitHub
   TLS chain). A one-paragraph "the hosted endpoint serves the
   same content as the GitHub release; verify SHA256SUMS for
   stricter integrity" note belongs in the guide before the
   hosted endpoint lands.

5. **Windows symmetry.** The PR adds a Unix `install` alias but
   doesn't add a corresponding Windows alias (e.g. `install.ps1`
   or `install.cmd`). If the hosted endpoint contract is "one URL
   per OS", Windows users get a worse experience by default. If
   the contract is "Unix only at the hosted URL, Windows users use
   `install-qwen.bat` from the GitHub release", the doc should
   say so explicitly so Windows users don't paste the `curl | bash`
   line into PowerShell.

## Optional nits (not blocking)

- The release-asset-config comment "keep byte-for-byte identical"
  could be enforced at config-build time by hashing both source
  paths after read and asserting equality before writing
  `install`, rather than relying on the test to catch drift after
  the fact.
- The `INSTALLATION_ASSET_NAMES` ordering at test line 327 puts
  `install` between `install-qwen.sh` and `install-qwen.bat`,
  which doesn't match alphabetical or by-platform grouping — a
  minor `sort()` on the array would future-proof against
  ordering churn.
