# block/goose PR #8972 — feat(docs): add Voidly Pay MCP extension

- Author: EmperorMew
- Head SHA: `016908899006bf393b1e84a628f3276bd3247542`
- Diff: +171 / -0, single new file `documentation/docs/mcp/voidly-pay-mcp.md`

## Observations

1. **`voidly-pay-mcp.md:1-6` frontmatter** uses `title` / `description` only, matching the docusaurus convention used by sibling `cash-app-mcp.md` and `alby-mcp.md` (the closest analogs the author cites — both payment-rail MCPs shipped via `npx`). Imports for `Tabs`, `TabItem`, `CLIExtensionInstructions`, `GooseDesktopInstaller` mirror those files line-for-line. Structurally clean.
2. **`voidly-pay-mcp.md:18-26` Quick Install block** uses the `goose://extension?cmd=npx&arg=-y&arg=%40voidly%2Fpay-mcp&...` deep link for desktop and `npx -y @voidly/pay-mcp` for CLI. Matches the deep-link format used by other npm-based MCP entries. The `id`, `name`, `description` query params are URL-encoded correctly.
3. **`voidly-pay-mcp.md:30+` Configuration section** uses the `<GooseDesktopInstaller>` and `<CLIExtensionInstructions>` components with consistent `extensionId="voidly-pay"`. `customStep3` flags the keypair-on-disk model (`~/.voidly-pay-keypair.json`) and points users at `voidly.ai/pay/claim` for the 10-credit faucet — important because users will hit a 402 on the first paid call otherwise and need to know where to go.
4. **Concerns worth surfacing for maintainer review** — this PR introduces a third-party payment rail (x402, USDC on Base mainnet) into the goose docs. Three things a maintainer should weigh that aren't really a "code review" question: (a) goose's docs precedent for payment-rail MCPs (cash-app-mcp, alby-mcp exist, so there *is* precedent); (b) the contract address `0xb592...1c12` is referenced as Sourcify-verified and the PR cites a public proof-of-reserves page — those claims should be independently verified before merging since merging implies docs-level endorsement; (c) the "42 tools" claim and the marketplace-pricing examples should match the actual current `@voidly/pay-mcp@0.5.0` package output, otherwise the docs will rot the moment the package version changes.
5. **Build claim** — author says `yarn build` (Docusaurus) passed locally. Given the file uses several MDX components and external links (basescan, voidly.ai), the CI Docusaurus build will be the real check; verify it green before merging.

## Verdict: `needs-discussion`

The mechanical structure of the file is correct and consistent with the closest analog docs. What blocks straight approval is policy-level: this is a third-party paid-MCP docs entry that ties goose's documentation to a specific smart contract on Base and a specific off-chain credit system. A goose maintainer should confirm the project's stance on listing payment-rail MCPs (precedent exists but each new one is a small endorsement), and ideally have one human verify the contract address + reserve-proof claims before merge so the docs aren't a future liability.
