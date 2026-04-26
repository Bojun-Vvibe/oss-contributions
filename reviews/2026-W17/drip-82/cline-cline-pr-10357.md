# cline/cline PR #10357 — chore(deps): bump dompurify from 3.3.3 to 3.4.1 in /webview-ui

- Repo: cline/cline
- PR: #10357
- Head SHA: `df6851b9`
- Author: @app/dependabot
- Diff: +65/-5 across 2 files (`webview-ui/package.json`, `webview-ui/package-lock.json`)

## What changed

Patch-to-minor `dompurify` bump (`^3.3.3` → `^3.4.1`) in `webview-ui/package.json:30`. Lockfile gets the new `dompurify` version plus a surprising 60-line block of newly-bundled `@tailwindcss/oxide-wasm32-wasi/node_modules/{@emnapi/core,@emnapi/runtime,@emnapi/wasi-threads,@napi-rs/wasm-runtime,@tybys/wasm-util,tslib}` `inBundle: true` entries — none of which have anything to do with `dompurify`.

## Specific observations

- `dompurify` 3.4.0 added the `RETURN_DOM_IMPORT` config option and changed `DOMPurify.removeAllHooks()` to actually remove hooks added via `addHook` (previously only removed `uponSanitizeElement` hooks). 3.4.1 fixed a regression where `Comment` nodes inside `<template>` were being stripped. Neither is a published security CVE but the hook-removal behavior change is observable — `git grep -nE "DOMPurify\.(addHook|removeAllHooks)" webview-ui/` will surface any cline code paths that registered `uponSanitizeAttribute` or `uponSanitizeShadowNode` hooks expecting the old "doesn't actually remove me" behavior.
- The lockfile diff at `webview-ui/package-lock.json:6240-6296` introduces six brand-new `@tailwindcss/oxide-wasm32-wasi/node_modules/*` `inBundle: true` entries (the WASM/WASI fallback variant of tailwind's native oxide compiler used when no native binary is available, e.g. on Termux/iSH/some CI runners). These are completely unrelated to `dompurify` and indicate the contributor ran a full `npm install` instead of `npm install --package-lock-only dompurify@3.4.1`. The bundled-dep noise is harmless at runtime (the `optional: true` flag at every entry means npm won't actually install them on platforms with a native binary) but it's lockfile churn that obscures review.
- The `webview-ui/package-lock.json:30` `dompurify` version bump is the only "real" change — everything else is collateral from the unscoped install. There's no `npm-shrinkwrap.json` so `npm ci` reproducibility on the bundled-deps entries depends on the registry mirror's tarball-hash stability. Two npm bumps in one drip cycle (this one + the tailwind oxide bundling) should not share a single PR title.

## Verdict

`needs-discussion`

## Rationale

The actual `dompurify` bump is fine in isolation — both 3.4.0 hook-removal and 3.4.1 template-comment fixes are net wins — but the PR ships with 60 lines of unrelated `@tailwindcss/oxide-wasm32-wasi` lockfile churn that should not have been included. Recommend either (a) ask dependabot to regenerate with `npm install --package-lock-only dompurify@3.4.1` so the diff is the single-line bump it should be, (b) split off the tailwind-oxide-wasm bundling as its own PR with its own justification, or (c) merge as-is only after a maintainer confirms `git grep "DOMPurify.removeAllHooks\|DOMPurify.addHook"` returns no callers that depend on pre-3.4.0 behavior. The bundled-deps churn is per-OS noise but conceals what changed.
