# sst/opencode PR #25538 — docs: sort docs sidebar entries by visual width

- **Author:** PanAchy
- **Head SHA:** `5e510a6303fc7bbe8d8f24e65bf0fd49b94df176`
- **Base:** `dev`
- **Size:** +3 / -3 (1 file)
- **Closes:** #25536

## What it does

Reorders top-level docs sidebar entries in `packages/web/astro.config.mjs` so the
new sequence visually steps from short → long labels: Intro, Config, Network,
Windows, Providers, Enterprise, Troubleshooting.

## Diff observations

`packages/web/astro.config.mjs:174-205` — the only change moves `"providers"`,
`"enterprise"`, `"troubleshooting"` from the front of the `sidebar` array to
after the `Windows` / `windows-wsl` entries:

```diff
       sidebar: [
         "",
         "config",
-        "providers",
         "network",
-        "enterprise",
-        "troubleshooting",
         {
           label: "Windows",
           ...
         },
         {
           ...
           link: "windows-wsl",
         },
+        "providers",
+        "enterprise",
+        "troubleshooting",
```

This matches the screenshot in the PR description. No content changes; pure
ordering.

## Risk

Cosmetic. Worth a sanity check that nothing else (search index, redirects,
in-page TOC) hard-codes the previous order, but a quick grep-style review of
the file shows the array is the single source for that nav slice.

## Nits (non-blocking)

- The PR title says "by visual width" but the result is a partial ordering
  driven by the new pages plus an existing Windows group. Future readers
  shouldn't expect a strict width sort to be enforced anywhere.

## Verdict

**merge-as-is**
