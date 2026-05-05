# sst/opencode #25841 — fix: Respect OTEL variables

- **Head SHA:** `bfd17ebd20189110da80dd24917287ea60dfbc80`
- **Base:** `dev`
- **Author:** maxkomarychev (Max Komarychev)
- **Size:** +8 / −4 across 1 file (`packages/core/src/effect/observability.ts`)
- **Closes:** #25839
- **Verdict:** `merge-after-nits`

## Summary

Fixes a regression where `resource()` in the OTEL setup was hard-coding
`serviceName: "opencode"` and `serviceVersion: InstallationVersion`,
silently overriding any `OTEL_SERVICE_NAME` / `service.version` /
`deployment.environment.name` the user had set via env or
`OTEL_RESOURCE_ATTRIBUTES`. Also stops `deployment.environment.name`
from being clobbered by `InstallationChannel` when the user supplied
their own.

## What's right

- **Precedence is now correct.** New behavior at
  `packages/core/src/effect/observability.ts:42` reads
  `process.env.OTEL_SERVICE_NAME ?? attributes["service.name"] ??
  "opencode"` — i.e. explicit env beats parsed
  `OTEL_RESOURCE_ATTRIBUTES` beats hard-coded default. This is the
  precedence OpenTelemetry SDK conventions specify (`OTEL_SERVICE_NAME`
  is documented as taking priority over `service.name` set via
  `OTEL_RESOURCE_ATTRIBUTES`), so the change aligns with spec.

- **Destructure-then-spread pattern is the clean fix.** The old code
  did `...attributes` and then unconditionally re-set
  `"deployment.environment.name": InstallationChannel`, which silently
  won over any user value. The new code destructures the three
  user-controllable keys out of `attributes` (`service.name`,
  `service.version`, `deployment.environment.name`) and spreads only
  `remainingAttributes`, then re-applies the deployment env with a
  proper `??` fallback (`observability.ts:51-52`). This means future
  additions to `attributes` won't accidentally collide with the
  hard-coded resource fields.

- **`serviceVersion` fix is consistent.** Previously
  `InstallationVersion` always won; now it falls back only when the
  user did not provide `service.version` (`observability.ts:43`).
  Symmetric with the service-name fix.

- **Manual verification is real.** The PR description shows the author
  ran with `OTEL_SERVICE_NAME=my-service` and
  `OTEL_RESOURCE_ATTRIBUTES="...deployment.environment.name=my-env"`
  and confirmed values landed in Jaeger correctly. That's exactly the
  failure mode being fixed.

## Nits / before-merge

1. **No env-var fallback for `service.version` or
   `deployment.environment.name`.** The PR respects `OTEL_SERVICE_NAME`
   directly from `process.env`, but for the other two it relies on
   them already having been parsed into `attributes` (presumably from
   `OTEL_RESOURCE_ATTRIBUTES`). That's fine for users who use
   `OTEL_RESOURCE_ATTRIBUTES`, but the SDK also honors the dedicated
   env vars `OTEL_SERVICE_VERSION` (less common) and there is no
   dedicated env var for deployment environment — so this is mostly
   fine. Worth a one-line comment noting the asymmetry, though, to
   prevent the next contributor from "fixing" it inconsistently.

2. **Underscore-prefixed unused destructure names trip some
   linters.** `const { "service.name": _n, "service.version": _v,
   "deployment.environment.name": _env, ...remainingAttributes }`
   (`observability.ts:44`) — depending on the eslint config, `_n` and
   `_v` may still emit `no-unused-vars` if the rule isn't configured
   with `argsIgnorePattern: "^_"` for destructure. CI presumably
   passes, but worth checking.

3. **No test coverage added.** The fix is a pure precedence change;
   `resource()` is presumably exercised in some OTEL integration test,
   but a 3-case unit test (env wins / attributes win / default wins)
   would lock in the precedence so it doesn't regress again. Not a
   blocker for this size of fix, but a follow-up worth filing.

## Risk

Low. The diff is mechanical, the behavior change is exactly the bug
fix described, and there are no additional code paths affected. Only
realistic regression risk: a user previously *relying* on the old
override behavior — but per the issue (#25839), users were
specifically complaining that the override was *broken*, so anyone
depending on it would have already been working around it.

## Verdict

`merge-after-nits` — the nits are documentation/follow-up only; the
code change itself is correct. Merge as-is is also defensible.
