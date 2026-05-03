# sst/opencode PR #25559 — chore: remove tree-sitter-powershell from trustedDependencies

- Head SHA: `cce5e15f628b`
- Closes: #25563
- Files changed:
  - `bun.lock` (-1)
  - `package.json` (-1)

## Analysis

Removes `tree-sitter-powershell` from the root `trustedDependencies` array in both `bun.lock` and `package.json`. Bun's `trustedDependencies` controls which packages are allowed to run lifecycle scripts (`postinstall`, etc.) at install time.

Spot diff:
- `bun.lock:665-672` strips the entry from the lock-file's `trustedDependencies` block.
- `package.json:121-126` strips the matching string from the source manifest, keeping `protobufjs`, `tree-sitter`, `tree-sitter-bash`, `web-tree-sitter`, `electron`.

The justification (per PR body) is that the package is consumed exclusively via its bundled WASM artifact:
```
await import("tree-sitter-powershell/tree-sitter-powershell.wasm" as string, { with: { type: "wasm" } })
```
…in `packages/opencode/src/tool/shell.ts`. Because runtime consumption is WASM-only, the native `.node` binding produced by the package's `postinstall` is never loaded, so granting it lifecycle-script trust just executes a build step we then throw away — it's pure install-time cost (and supply-chain surface area).

Risks:
- If a future code path tries to `require("tree-sitter-powershell")` for the native binding, it will now fail at runtime because the postinstall did not build the `.node` artifact. Easy to catch in CI, but worth flagging in a follow-up `// WASM-only` comment near the import in `shell.ts`.
- The two siblings `tree-sitter-bash` / `web-tree-sitter` / `tree-sitter` remain trusted; a quick consistency audit would confirm whether any of them are also WASM-only and could follow the same diet. Not in scope for this PR.

## Verdict

`merge-as-is`

## Nits

- Add a one-line comment above the WASM import in `packages/opencode/src/tool/shell.ts` noting "WASM-only consumer; do not re-add to trustedDependencies" so the next person to touch it understands the constraint.
