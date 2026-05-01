# BerriAI/litellm #27007 — fix(auth): block missing write routes for proxy admin viewers

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/27007
- **HEAD SHA:** `c27951b53b6fe6f0581e4309d545832c3cea5e0b`
- **Author:** stuxf
- **Verdict:** `merge-after-nits`

## What the diff does

Closes a fall-through hole in the `PROXY_ADMIN_VIEW_ONLY` route
blocklist. The prior implementation hand-listed write routes inline
inside an `elif` branch in `_check_proxy_admin_viewer_access`, and
because the surrounding logic falls through to "allow" on no match,
any management write route not explicitly named was silently
permitted to view-only admins. Audit found the following missing:

- `/team/block`, `/team/unblock`
- `/team/permissions_update`, `/team/permissions_bulk_update`
- `/jwt/key/mapping/{new,update,delete}`
- `/key/bulk_update`
- `/key/{key_id}/regenerate` and `/key/{key_id}/reset_spend` (templated paths)

Fix at `litellm/proxy/auth/route_checks.py`:

1. New module-level `_PROXY_ADMIN_VIEW_ONLY_BLOCKED_ROUTES` frozenset
   (`:18-50`) lists every blocked write route, including the eight
   newly-added entries. Includes a header comment ("Adding a new
   write endpoint to a management router REQUIRES adding it here too
   — the surrounding check falls through to 'allow' if the route is
   not matched") that names the architectural footgun.
2. New `_PROXY_ADMIN_VIEW_ONLY_BLOCKED_KEY_SUFFIXES = ("/regenerate",
   "/reset_spend")` (`:55`) handles the `/key/{key_id}/...`
   path-parameterized routes that the literal-string set can't match.
3. The inline list at the call site (`:629-651`) collapses to
   `route in _PROXY_ADMIN_VIEW_ONLY_BLOCKED_ROUTES or
   (route.startswith("/key/") and route.endswith(_..._SUFFIXES))`.
4. Where the blocklist references key-management routes, it pulls
   from the `KeyManagementRoutes` enum
   (`KeyManagementRoutes.KEY_GENERATE.value`, etc.) so adding a new
   member to that enum *also* adds it to the blocklist (well,
   conceptually — see nit 1).

`tests/test_litellm/proxy/auth/test_route_checks.py:93-156` —
`test_proxy_admin_viewer_blocked_management_writes` parametrizes
over 14 blocked routes (mix of newly-blocked and previously-blocked
baseline), each asserting `HTTPException(403)` with the route name in
the detail; `test_proxy_admin_viewer_allowed_management_reads`
parametrizes over 12 allowed read routes asserting no exception.

## Why it's right

The architectural failure mode of the prior code was "allowlist by
omission": the inline check matched a hand-rolled list and fell
through to allow on miss, so the security property (view-only admins
can't write) silently degraded any time a new write route shipped
without being added here. The fix doesn't change the fall-through
semantics, but it (a) makes the list easier to audit by hoisting it
to a named module-level frozenset, (b) reduces drift risk by
referencing `KeyManagementRoutes` enum values for the key write
routes, and (c) names the footgun in a header comment so the next
person adding a write endpoint sees the requirement.

The split between literal routes (`_PROXY_ADMIN_VIEW_ONLY_BLOCKED_ROUTES`)
and suffix patterns (`_PROXY_ADMIN_VIEW_ONLY_BLOCKED_KEY_SUFFIXES`)
is the right shape — `frozenset` membership for O(1) literal lookup,
suffix-tuple for the path-parameterized `/key/{key_id}/...` cases.
`str.endswith(tuple)` is the idiomatic Python way to test a set of
suffixes in one call.

The test parametrization is dispositive: each newly-blocked route
gets its own failing assertion line, and the parallel
`allowed_management_reads` test locks the *negative* property that
view-only admins are still allowed to read. Without the negative
test a future "let's just block everything" change could pass the
positive parametrization but break legitimate read access.

## Nits

1. **The "use the enum" pattern is incomplete.** Key write routes
   are pulled from `KeyManagementRoutes.KEY_*.value`, but team /
   user / model / JWT routes are still string literals. Either all
   should be enum-backed (consistent) or none should be (consistent),
   but the half-and-half state is the worst of both worlds — a
   reader sees the key routes use the enum and assumes that's the
   pattern, then misses that team routes don't. Two paths: (a)
   move all write-route literals into a single enum
   (`ManagementWriteRoutes`?) referenced from both the router and
   this blocklist, or (b) drop the enum reference and use bare
   strings everywhere with an inline `# keep in sync with...`
   comment.

2. **No CI guard against the actual drift mode.** The header comment
   at `:18-21` correctly identifies the failure ("adding a new
   write endpoint REQUIRES adding it here too") but there's no
   automated check that any route in the management router *is*
   either in the blocklist or explicitly opted-out as a read
   route. A test that loads the actual FastAPI router and asserts
   "every route registered as POST/PUT/DELETE on
   `/team`, `/user`, `/key`, `/model`, `/jwt/key/mapping` is in
   `_PROXY_ADMIN_VIEW_ONLY_BLOCKED_ROUTES`" would catch the next
   drift before code review.

3. **`/v2/team/list` test at `:144`** — good coverage, but no
   corresponding `/v2` write routes in the blocklist. If `/v2`
   shadow routes exist for any blocked write, those need to be in
   the set too. Worth confirming via `grep -rn 'v2.*team' litellm/proxy/`.

4. **Suffix matching at `:677`** — `route.endswith(("/regenerate",
   "/reset_spend"))` will match `/key/foo/bar/regenerate` if such a
   route ever exists. That's the conservative direction (block
   more rather than less) and is fine for a blocklist, but worth
   a comment so a future reader doesn't widen the suffix list to
   something that could over-block reads.

5. **`_PROXY_ADMIN_VIEW_ONLY_BLOCKED_KEY_SUFFIXES` is a tuple, not
   a frozenset** — `str.endswith` requires a tuple, so this is
   correct, but the inconsistency with the literal frozenset
   above could confuse a reader. One-line comment explaining
   "tuple required by str.endswith" would help.

6. **Test detail string assertion** at `:135`
   (`assert blocked_route in str(exc_info.value.detail)`) is
   robust against the exact wording of the 403 message but is
   weaker than asserting the message *also* says "PROXY_ADMIN_VIEW_ONLY"
   somewhere — without that, a future change that 403s for a
   different reason (e.g. CSRF) on the same route would still
   pass the test.

## Verdict rationale

Right fix shape (hoist the inline list, reference enum where it
exists, name the architectural footgun in a comment), right test
surface (parametrized blocked + allowed). Half-and-half enum vs.
literal pattern is the worst-of-both-worlds outcome and would benefit
from one direction or the other; missing CI guard against the actual
drift mode the comment warns about.

`merge-after-nits`
