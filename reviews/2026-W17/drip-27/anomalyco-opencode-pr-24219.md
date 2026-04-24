# anomalyco/opencode PR #24219 — Generated HTTP route inventory

- **Repo:** anomalyco/opencode
- **PR:** [#24219](https://github.com/anomalyco/opencode/pull/24219)
- **Head SHA:** `2b028287e21ae5931d6654738872c7b5ba9bc55a`
- **Author:** kitlangton
- **Size:** +286 / −23 across 3 files
- **Reviewer:** Bojun (drip-27)

## Summary

Adds a Bun script that introspects the actual Hono route registrations
across opencode's HTTP surfaces, deduplicates, and emits a Markdown
inventory. Checks in the generated `specs/effect/http-route-inventory.md`
and updates the existing HttpApi migration plan to reference the
generated table and reclassify the workspace surface as "bridged
partial".

## Key changes

### `packages/opencode/script/http-route-inventory.ts:1–138` — new file

Bun script. Imports the actual route surface modules
(`InstanceRoutes`, `ControlPlaneRoutes`, `WorkspaceRoutes`,
`UIRoutes`) plus their path constants, walks each Hono app, and
emits a sorted Markdown table.

The script defines two explicit allowlist/denylist sets:

- `bridged: Set<string>` — routes confirmed to have a v2/Effect
  bridge today (file/mcp/permission/question/config/etc.).
- `topLevelNext: Set<string>` — routes scheduled for next
  migration round (path/vcs/command/agent/skill/lsp/formatter).

Anything in the registered route set that isn't in `bridged` and
isn't in `topLevelNext` falls through to a default "not migrated"
status in the generated table.

```ts
const bridged = new Set([
  key("GET", "/question"),
  key("POST", "/question/:requestID/reply"),
  key("POST", "/question/:requestID/reject"),
  key("GET", "/permission"),
  key("POST", "/permission/:requestID/reply"),
  key("GET", "/config"),
  key("GET", "/config/providers"),
  key("GET", "/provider"),
  key("GET", "/provider/auth"),
  key("POST", "/provider/:providerID/oauth/authorize"),
  key("POST", "/provider/:providerID/oauth/callback"),
  key("GET", "/project"),
  key("GET", "/project/current"),
  key("GET", FilePaths.list),
  key("GET", FilePaths.content),
  key("GET", FilePaths.status),
  key("GET", McpPaths.status),
  ...Object.values(WorkspacePaths).map((path) => key("GET", path)),
])
```

Importing the actual `*Paths` constants from the source modules
(rather than hard-coding string literals) is a nice touch — if a
path constant moves, the inventory generator picks up the new
value automatically and stays in sync.

### `key(method, path)` and `normalize(prefix, route)` helpers

Lines 76–82. `key` is a tiny string concatenator for the dedup
sets. `normalize` joins a prefix and a route, with two
correctness details:

```ts
function normalize(prefix: string, route: string) {
  if (!prefix) return route
  if (route === "/") return prefix
  return `${prefix}${route}`.replaceAll(/\/+/g, "/")
}
```

The `route === "/"` early return is the right shape — without it
a `Hono` mounted at `/instance` registering `/` would render as
`/instance/`, which is a different URL from `/instance` to most
HTTP clients. The trailing `replaceAll(/\/+/g, "/")` collapses any
accidental double slashes from concatenation.

### `routes(surface, app, prefix)` walker

The actual recursive walker into Hono's route table. (Full body
not in the diff sample I have, but the signature and the way
`prefix` threads through is consistent with Hono's `app.routes`
shape.)

### `specs/effect/http-route-inventory.md` (generated)

Checked-in artefact. The point of checking in a generated file
is reviewer-friendliness: PRs that touch routes will show the
inventory diff alongside the route diff, so reviewers can see at
a glance whether a route was added/removed/migrated without
running the script themselves.

### Migration-plan doc update

Updates the existing HttpApi migration-plan doc to reference the
generated inventory and to reclassify the workspace surface from
whatever it was before to "bridged partial". Small textual change
to a planning doc, low risk.

## What's good

- Importing the live route modules and walking them is the
  fundamentally correct shape for a generator. Hand-maintained
  inventories drift; generated ones don't.
- Importing the `*Paths` constants by reference (rather than
  hard-coding the string literals into the generator's allowlist)
  means the allowlist follows the canonical path definitions
  automatically. Renaming or restructuring path constants doesn't
  break the inventory or require dual updates.
- Two explicit sets (`bridged`, `topLevelNext`) is honest about
  the migration state — every route is either confirmed migrated,
  scheduled for the next round, or implicitly not-yet-migrated.
  No route can quietly get "lost" between categories because the
  default fall-through is the most pessimistic state.
- Checking in the generated `.md` puts the migration progress
  into PR review surface and into git history. Future PRs that
  add routes will visibly grow the inventory; PRs that migrate
  routes will visibly flip statuses. Both are exactly what you
  want for a multi-quarter migration.
- `normalize` correctly handles the `route === "/"` and
  double-slash cases that are the usual sources of "the route
  table says `/foo/` but curl 404s on `/foo`" confusion.

## Concerns

1. The script has no test. For a generator script this is usually
   fine (the output *is* the test, in the form of the checked-in
   `.md`), but a small assertion that "the generator's output for
   a known fixture set matches a known string" would catch the
   case where someone refactors `routes()` and silently changes
   the sort order or the path normalization.

2. The two `Set<string>` allowlists will need to be edited in this
   file every time a route migrates. There's no CI check (visible
   in the diff sample) that the sets are consistent with the
   actual migration code — i.e. if someone marks a route as
   `bridged` in this script but never actually wires the v2/Effect
   bridge, the inventory will lie. Worth a follow-up: have the
   generator *verify* that every entry in `bridged` resolves to a
   v2 handler, and fail loud if it doesn't.

3. `topLevelNext` enumerates the next migration round inline, in
   *this* file. That couples the generator to the migration
   roadmap in a way that means the generator has to be touched
   on every roadmap change. An alternative would be to have the
   roadmap doc itself emit machine-readable hints the generator
   reads. Not worth blocking on for the first version of the
   generator, but a refactor target for v2.

4. The script imports a bunch of modules at top level
   (`InstanceRoutes`, `ControlPlaneRoutes`, etc.). If any of those
   modules has top-level side effects (e.g. opening a DB, starting
   a worker), running the generator would also trigger those. For
   a Bun script this is usually safe because Bun is fast to spin
   up and tear down, but worth confirming the imported modules
   are pure-on-import.

5. No CI integration (in the diff sample) for "fail the build if
   the generated inventory is out of date relative to the routes".
   Without that, the inventory is a snapshot that will drift the
   moment someone adds a route in another PR and forgets to rerun
   the generator. A `bun run script/http-route-inventory.ts >
   /tmp/x && diff -u specs/effect/http-route-inventory.md /tmp/x`
   step in CI would close the loop.

## Risk

Very low for runtime: the script is a developer-facing tool, not
shipped at runtime, and the generated `.md` is documentation. The
only runtime-adjacent change is the migration-plan doc, which is
a status reclassification.

## Verdict

**merge-after-nits**

The shape of the solution is right. Concern 5 (CI freshness check)
is the only one with real long-term cost — without it, the
generated inventory will drift and stop being trustworthy within
a few PRs. Concern 2 (allowlist verification against actual v2
wiring) is a strong follow-up but can land separately. Other
concerns are forward-looking polish.

## What I learned

"Generate the doc by walking the live registry, then check in the
generated artefact and gate CI on freshness" is the right pattern
for migration-progress tracking. It beats both "hand-maintain a
status doc" (drifts immediately) and "publish a dashboard" (out
of band, easy to ignore in PR review). The trick is making the
generator's allowlist *also* drift-resistant, which this PR mostly
does by importing path constants by reference but leaves the
"bridged vs not" classification as inline editing — that's the
remaining drift risk.
