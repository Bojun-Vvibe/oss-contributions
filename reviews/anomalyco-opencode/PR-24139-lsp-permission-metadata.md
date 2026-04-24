# PR #24139 — include LSP request details in permission metadata

**Repo:** sst/opencode • **Author:** simonklee • **Status:** open
• **Net:** +12 / −3

## Change

`LspTool` was calling `ctx.ask({ permission: "lsp", patterns: ["*"],
always: ["*"], metadata: {} })` — empty metadata meant the permission
prompt UI had nothing to render. This PR fills `metadata` with
`{ operation, filePath, line, character }` so prompts can describe
the request before approval. Also adds `.pipe(Effect.orDie)` to the
Effect-returning closure to match the other tool implementations.

## Load-bearing risk

Two separate changes hidden in a small diff:

1. **Metadata exposure.** `filePath` here is the *resolved absolute
   path* (after `path.isAbsolute` + `path.join(Instance.directory,
   ...)`) — that's good for readability but means the prompt now
   leaks the user's full home-dir path into wherever permission
   metadata gets logged or persisted (audit log, telemetry, session
   replay). For most users this is fine; for shared/recorded sessions
   it's a small privacy regression. Consider rendering relative-to-
   instance-dir in the metadata, with the absolute path only kept
   for the actual call.

2. **`Effect.orDie` change.** This converts any error in the inner
   `Effect.gen` into a defect (panic) rather than a typed failure.
   The PR justifies this as "match the other tool implementations,"
   but `LspTool` may have been deliberately keeping recoverable
   errors typed — e.g. an LSP server crash or a `position out of
   range` should arguably produce a tool-call error result the model
   can react to, not kill the fiber. Need to verify which behavior
   the LSP service actually surfaces; if it raises typed failures
   today, `orDie` swallows the nuance.

The existing `patterns: ["*"], always: ["*"]` is unchanged, so the
permission grant *scope* is still a single global rule per LSP tool
call. That's a separate (pre-existing) sharp edge: a user who once
clicks "always allow" for LSP can never scope it to a single file or
operation again. This PR doesn't fix that, but fattening the metadata
makes that decision feel more granular than it actually is — users
may believe an "always" click only covers the path shown.

## Concrete next step

Split the diff: land the metadata addition standalone, then justify
`.pipe(Effect.orDie)` separately with a test case that shows an LSP
server failure today vs. after. For the metadata itself, add a
display path (relative-to-instance) in addition to the absolute path,
and update the permission UI's "always" copy to clarify it grants
all-LSP-calls, not "this file" — otherwise the new metadata is
actively misleading.

## Verdict

Useful UX improvement bundled with a quiet error-handling change
that deserves its own justification.

## What I learned

"Match the other tools" is a smell when applied to error policy —
each tool's failure modes are different, and the temptation to
homogenize via `orDie` trades typed recovery for symmetry. The
"add metadata to a permission prompt" pattern also creates an
expectation gap: users will read the prompt as scoping the grant.
