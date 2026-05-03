# sst/opencode PR #25571 — docs(opencode): clarify C# LSP requires csharp-ls

- **Author:** sulsDotK
- **Head SHA:** `14e84e9ecb4abe38994c445d5bc5fde4e80c4e3f`
- **Base:** `dev`
- **Size:** +1 / -1 (1 file)

## What it does

Updates the built-in LSP requirements table in
`packages/web/src/content/docs/lsp.mdx` so the C# row mentions that `csharp-ls`
must be available on PATH, not just `.NET SDK`. The runtime already launches
`csharp-ls` for `.cs`/`.csx`, so this is a doc-vs-behavior fix.

## Diff observations

`packages/web/src/content/docs/lsp.mdx:19`:

```diff
-| csharp             | .cs, .csx                                                           | `.NET SDK` installed                                         |
+| csharp             | .cs, .csx                                                           | `.NET SDK` installed; csharp-ls command available            |
```

Single-row change, table layout preserved (column widths look untouched
visually; pipe alignment is fine even if it's not strictly column-aligned in
source).

## Strengths

- Matches the existing convention used by neighboring rows
  (`clojure-lsp command available`, `dart command available`).
- Pure docs change, zero code risk.

## Nits (non-blocking)

- Could backtick `csharp-ls` for typographic consistency with the other rows
  (e.g. `` `csharp-ls` command available``). Trivial, fine to land as-is.
- Optional follow-up: a one-line install hint
  (`dotnet tool install -g csharp-ls`) would save a search, but the PR body
  notes the runtime already surfaces that message.

## Verdict

**merge-after-nits** (the backtick consistency nit; otherwise ready)
