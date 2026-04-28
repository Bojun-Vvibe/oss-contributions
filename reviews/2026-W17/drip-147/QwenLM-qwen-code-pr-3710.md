# QwenLM/qwen-code #3710 — feat(cli): customize banner area (logo, title, hide)

- PR: https://github.com/QwenLM/qwen-code/pull/3710
- Head SHA: `d2d751e32116d9d7521f5d63e4692693ba7dbc9d`
- Author: chiga0 (ChiGao)
- Files: 10 files, +2001/−8 (590-line design doc EN, 556-line zh-CN, 351-line resolver, 333-line resolver tests, 43-line schema, 35-line Header, 18-line AppHeader test, 46-line Header test, 14-line AppHeader, 15-line vscode schema)
- Implements design from #3671, tracks #3005

## Observations

- Three opt-in `ui.*` settings: `hideBanner` (drop logo + info panel), `customBannerTitle` (replace `>_ Qwen Code` title; version suffix is *always* appended), `customAsciiArt` (replace QWEN ASCII logo; accepts string, `{path}`, or `{small, large}` for width-aware tiers).
- **The right design choice that earns trust**: operational signals are **locked**. Version suffix, auth + model line, working directory cannot be hidden by user config. PR cites the design rationale: "critical for support and security posture." This is the correct invariant — banner customization is brand-skin only, not a vector for hiding which Qwen build/model/auth is actually running. Closes the support-can't-debug-the-screenshot risk and the social-engineering "make it look like a different tool" risk.
- Resolver at `packages/cli/src/ui/utils/customBanner.ts:1-351` does per-scope normalization so each `{path}` resolves against the file that declared it (workspace `.qwen/` vs. user `~/.qwen/`). Correct shape — declaring scope determines resolution scope, not process cwd. Reads use `O_NOFOLLOW` and a 64 KB cap on POSIX (Windows: plain read-only). The `O_NOFOLLOW` is the load-bearing primitive — it closes the symlink-attack vector where a hostile workspace points `path: "~/.ssh/id_rsa"` at a target outside its declaring scope.
- Banner-specific sanitizer strips OSC/CSI/SS2/SS3 sequences and other C0/C1 controls **while preserving `\n`**, then caps art at 200 lines × 200 cols and titles at 80 characters. The sanitizer is the **second load-bearing primitive** — without it, `customBannerTitle: "\x1b[31mhostile"` would paint terminal text red on every startup (a low-grade UX vandalism vector at minimum, possibly a phishing vector if combined with an extracted-from-screenshot-looks-genuine claim). The 200×200 / 80-char caps prevent the third class of vandalism — "boot a 1000-line ASCII bomb that screws up scrollback."
- Soft-fail discipline on every error path: malformed input, missing file, oversize file, non-regular file, sanitization rejection → `[BANNER]` warn + fall back to bundled QWEN logo / default title. **CLI never crashes on a banner config error.** Right shape — a cosmetic config never blocks the actual tool from starting.
- `<Header />` picks the widest custom tier that fits via shared `pickAsciiArtTier` helper, falls back to `shortAsciiLogo` otherwise. `<AppHeader />` extends the existing `showBanner` gate to honor `hideBanner` alongside the screen-reader fallback. Width-aware tiering (`{small, large}`) is the right shape — narrow terminals get a sensible default rather than a wrapped-and-mangled wide logo.
- `customAsciiArt` and `customBannerTitle` set `showInDialog: false` because a multi-line ASCII editor in the TUI is its own project. Power users edit `settings.json`. `hideBanner: true` mirrors `hideTips`. Correct scope discipline — don't ship a TUI editor for a feature 0.1% of users will use; let them edit JSON.
- Test surface: 4648 tests pass in `packages/cli` including 24 new resolver tests + 5 new Header/AppHeader tests. tsc clean, eslint clean.

## Risks / nits

- Resolver hits filesystem on every startup when `customAsciiArt` uses `path`. 64 KB cap + `O_NOFOLLOW` is fast enough that this is fine, but worth a single-run cache for the case where Header re-mounts during the React lifecycle (otherwise small race on rapid re-render).
- Sanitizer strips OSC (`\x1b]...\x07` or `\x1b]...\x1b\\`) — the terminator state machine is notoriously bug-prone. Worth fuzzing the sanitizer against malformed-OSC corpus seeds. The 333-line test file should include a "malformed OSC partial sequence" cell pinning that the sanitizer doesn't get stuck.
- `customBannerTitle: "Acme CLI"` produces `Acme CLI v0.15.3` (version always appended). What about RTL languages or wide-char titles? `80 chars` cap is char-count, not byte-count or display-width — a title of 80 East Asian wide chars renders as 160 columns, may overflow narrow terminals. Worth a `wcwidth`-aware cap.
- The 590+556 = 1146 lines of design doc are great for governance but make the diff hard to scan for actual code change. Worth landing the doc as a separate commit so reviewers can split-screen the implementation against the spec.
- `customAsciiArt: { small: ..., large: { path: ... } }` — mixing inline string and `{path}` between tiers should work, but the resolver's per-tier dispatch needs a clean shape. Verify the resolver tests cover the heterogeneous-tiers case.
- Three of five reviewer manual-smoke checkboxes still unchecked. Author should knock those off before requesting review.
- `showInDialog: false` for `customAsciiArt`/`customBannerTitle` correctly avoids building a TUI editor — but a discoverability cost. Worth a `qwen banner --help` subcommand surfacing the doc inline. Follow-up.
- PR is currently a **draft** until #3671 merges. Verify rebase plan is "drop the design-doc commits since they're already on main" not "force-push and re-add them."

## Verdict: `merge-after-nits`

**Rationale:** Excellent design discipline — locked operational signals (version/auth/model/cwd can't be hidden), correct security primitives (`O_NOFOLLOW` + scope-aware path resolution + escape-sequence sanitizer with size caps), soft-fail-on-config-error so banner bugs never block CLI startup, and `showInDialog: false` correctly defers a TUI editor to "users who want this can edit JSON." Pre-merge nits: (a) `wcwidth`-aware char cap for wide-char titles, (b) malformed-OSC corpus in resolver tests, (c) verify heterogeneous `{small: string, large: {path}}` cell exists, (d) finish the 5 manual-smoke checkboxes, (e) consider splitting the 1146-line design doc into a separate commit for easier diff scanning. Sane defaults, paranoid security, honest scope.
