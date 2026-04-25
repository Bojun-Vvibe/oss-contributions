# ollama/ollama#15703 — gemma4: allow reserved JSON Schema keys as parameter names

- PR: https://github.com/ollama/ollama/pull/15703
- Author: mverrilli (Michael Verrilli)
- +36 / -4
- Head SHA: `62b82e7f2d6f8c1b3fe776a1ccd43adf1389e8db`

## Summary

Fixes #15670 in the Gemma4 renderer: any tool parameter named `description`,
`type`, `properties`, `required`, or `nullable` was silently dropped from
the rendered tool declaration. Root cause is that
`writeSchemaProperties` called `isSchemaStandardKey()` on parameter *names*
in its outer loop — but those strings, while reserved as JSON Schema
keywords, are also entirely valid user-defined parameter names. The fix
removes the guard from the outer loop and instead applies it only at the
one nested call site that passes a raw schema object (rather than a
properties map) into `writeSchemaProperties`, by pre-filtering into a
local `filtered` map.

## Specific findings

- `model/renderers/gemma4.go:332` (SHA
  `62b82e7f2d6f8c1b3fe776a1ccd43adf1389e8db`) — the `if
  isSchemaStandardKey(name) { continue }` guard is removed from the outer
  property-name loop. This is the correct loop to remove from: every key
  iterated here is a user-declared parameter name, not a JSON Schema
  keyword.
- `model/renderers/gemma4.go:411` — the surviving filtering happens at the
  one nested-properties call site:
  ```go
  filtered := make(map[string]any, len(prop))
  for k, v := range prop {
      if !isSchemaStandardKey(k) {
          filtered[k] = v
      }
  }
  r.writeSchemaProperties(sb, filtered)
  ```
  This is the path where `prop` is a raw schema object (so its top-level
  keys *are* JSON Schema keywords like `properties`, `required`, etc.),
  and filtering them out before passing to `writeSchemaProperties` is the
  intended behavior. The fix is surgical: the guard moves from "always" to
  "only when the caller is handing me a raw schema."
- `model/renderers/gemma4_reference_test.go:639` —
  `reservedParamNamesTool()` declares a tool with three parameters
  (`propX`, `description`, `propY`) where `description` is in the middle
  of the `Required` list. The reference output at line 679
  (`reservedParamNamesDeclRef`) explicitly includes
  `description:{description:<|"|>The description<|"|>,type:<|"|>STRING<|"|>}`
  in the `properties:{...}` block and `<|"|>description<|"|>` in the
  `required:[...]` array. This is exactly the failure mode from the issue
  report (parameter present in `required`, missing from `properties`).
- The test is registered in `TestGemma4RendererMatchesReference` at
  line 1112, so it runs as part of the existing renderer reference suite.
  The PR description claims all 69 renderer tests pass with the fix.

## Verdict

`merge-as-is`

## Rationale

Correct root-cause fix at the right layer: the JSON Schema keyword guard
was applied at the wrong scope (parameter names, where it never should
have run) and is now applied only in the one nested-schema fallback where
keyword filtering is actually correct. The reference test reproduces the
exact failure mode from #15670 — a reserved-name parameter listed in
`required` but missing from `properties` — and the byte-level reference
string asserts the full surrounding declaration, so any regression here
would be caught immediately.
