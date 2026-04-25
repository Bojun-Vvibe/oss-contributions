# anomalyco/opencode#24229 — fix: lazy session error schema

**Verdict: merge-as-is**

- PR: https://github.com/anomalyco/opencode/pull/24229
- Author: Hona
- Base SHA: `5cd178ba7008969c9fc711c78603d7e3144b4ce8`
- Head SHA: `d704110e52ad243a20733cd3c8cf8a97bf3d6dbf`
- +4 / -1

## Summary

Tiny but load-bearing fix: defers `MessageV2.Assistant.fields.error`
access via `Schema.suspend(() => ...)` in the session `Event.Error`
schema definition to break a compiled-binary circular initialization
crash. Pairs with a 3-line addition to the Effect-Schema → Zod walker
(`packages/opencode/src/util/effect-zod.ts`) so the new `Suspend` AST
node is handled (otherwise the OpenAPI / event-payload generator
would `fail(ast)` on it and break BusEvent payload derivation).

## Specific findings

- `packages/opencode/src/session/session.ts:284` (head SHA
  `d704110e52ad243a20733cd3c8cf8a97bf3d6dbf`):
  ```ts
  // Reuses MessageV2.Assistant.fields.error (already Schema.optional) so
  // the derived zod keeps the same discriminated-union shape on the bus.
  // Schema.suspend defers access to break circular init in compiled binaries.
  error: Schema.suspend(() => MessageV2.Assistant.fields.error),
  ```
  The `Schema.suspend(() => ...)` thunk defers the field reference
  until first deserialize, which is the standard Effect-Schema fix
  for module-init cycles. The previous direct reference
  `error: MessageV2.Assistant.fields.error` evaluated at module load
  time — fine in `bun typecheck`, but in the AOT-compiled binary the
  `MessageV2` module's `Assistant` schema isn't fully initialized at
  the moment `Event.Error` is constructed, producing a TDZ-style
  crash. The PR description's verification command:
  ```
  bun -e "const session = await import('./src/session/session.ts');
          const { BusEvent } = await import('./src/bus/bus-event.ts');
          console.log(BusEvent.payloads().length, session.Event.Error.type)"
  ```
  is the right smoke test — it exercises both the lazy access and
  the BusEvent payload derivation that consumes the schema.
- `packages/opencode/src/util/effect-zod.ts:256`:
  ```ts
  case "Suspend":
    return z.lazy(() => walk(ast.thunk()))
  ```
  Maps `SchemaAST.Suspend` directly to `z.lazy()`, the Zod equivalent.
  Recursive `walk()` call resolves the inner schema on each access,
  preserving the discriminated-union shape that the upstream comment
  on the `error` field promises. Without this case, the `default:
  fail(ast)` arm would have been hit for any suspended schema in the
  bus event tree, breaking OpenAPI generation.
- The change correctly preserves the `Schema.optional` semantics —
  the suspended thunk returns `MessageV2.Assistant.fields.error`,
  which is already optional, so the discriminated-union shape on the
  bus stays identical. Worth confirming via a snapshot that the
  generated OpenAPI for the Error event looks the same before/after.

## Rationale

A four-line fix for a compiled-binary-only crash that doesn't show up
in `bun typecheck` (because TS evaluation order differs from the
runtime AOT init order). The `Schema.suspend` pattern is the
documented Effect-Schema escape hatch for exactly this class of
circular-reference problem, and the paired `effect-zod.ts` walker
update is necessary — otherwise the suspend would type-check fine but
crash the schema-to-zod conversion at runtime with `fail(ast)`. The
walker change uses `z.lazy(() => walk(ast.thunk()))`, which is the
canonical Zod equivalent and recursively re-enters `walk` so the
inner schema is processed by all the same arm logic as a direct
reference. The verification command in the PR body is well-chosen
because it loads both modules in sequence and reads
`BusEvent.payloads().length` — exercising the full
schema-to-payload-derivation path that previously crashed at startup.
The comment on the `error` field documents both the *what* (reuses
Assistant.fields.error to keep the bus discriminated-union shape)
and the *why* (Schema.suspend defers access to break circular init in
compiled binaries) — that's exactly the right level of comment for
a non-obvious workaround. No nits. Merge.
