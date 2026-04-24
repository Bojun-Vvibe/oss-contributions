# PR #19216 — Allow any granular approval requirement

**Repo:** openai/codex  
**Surface:** approval-policy schema in app-server-protocol; config parsing
in `codex-config`; `requirements.toml` allow-list type.  
**Severity:** Medium (security-adjacent: approval policy expressivity)

## What changed

`allowed_approval_policies` previously held concrete `AskForApproval`
values, requiring exact equality on the full policy object. The PR
introduces a tagged `ApprovalPolicyConstraint`:

```
ApprovalPolicyConstraint =
  | "untrusted" | "on-failure" | "on-request" | "never"     // exact match
  | { granular: GranularApprovalConstraint }                // shape match
```

with `GranularApprovalConstraint = "any" | GranularApprovalConfig`.
A wildcard is therefore *only* possible inside the `granular` family —
the author calls this out explicitly:

> This adds a scoped wildcard for the granular policy family only. It
> does not add a global `any` approval-policy escape hatch.

The schema is regenerated; TS fixtures land in the same PR; the
`configRequirements/read` API now returns the constraint shape verbatim
instead of collapsing it into one concrete granular instance.

## What's risky / wrong / missing

1. **Wildcard semantics deserve an explicit security note.** "Any
   granular policy" today means any combination of
   `mcp_elicitations`/`request_permissions`/`rules`/`sandbox_approval`/`skill_approval`.
   When a *new* granular sub-flag is added later
   (let's call it `network_approval`), pre-existing
   `{ granular = "any" }` allow-list entries will silently accept
   policies that disable that new flag. That is the correct meaning of
   "any" — but it means adding a new approval dimension is now a
   policy-loosening event for any deployment that uses this wildcard.

   This is a permanent governance trade-off and should be documented at
   the constraint definition site, not just in the PR body.

2. **The `GranularApprovalConfig` schema lists `request_permissions` and
   `skill_approval` with `"default": false`**, while
   `mcp_elicitations` / `rules` / `sandbox_approval` are required. That
   mismatch invites a constraint author to write a partial granular
   constraint and have it accept policies that toggle the defaulted
   fields freely. The `"any"` wildcard sidesteps this, but the
   non-wildcard `GranularApprovalConfig` constraint shape is now a
   second new feature with subtler semantics that the PR description
   doesn't fully cover.

3. **Backward compat of clients.** The schema change moves
   `allowedApprovalPolicies` items from `AskForApproval` to
   `ApprovalPolicyConstraint`. Any external client deserializing this
   field with a hand-rolled type will now break on the new variant.
   The PR regenerates TS fixtures, but downstream SDKs need a coordinated
   release. A note in the protocol changelog (or a version bump) is
   warranted given this is part of the v2 schema.

4. **Test naming hint of a gap.** The mentioned regression test is
   `map_requirements_toml_to_api_preserves_any_granular_approval_constraint`
   — that proves wildcard *plumbing* survives the round-trip, but does
   not prove that an `AskForApproval::Granular(_)` payload with a
   *future* unknown field is correctly accepted by the wildcard. A
   forward-compat test (deserialize a granular policy with an
   unmentioned field, then check it matches `granular = "any"`) would
   close the loop.

## Suggested fix

- Add an inline doc comment at the `GranularApprovalWildcard` definition
  explaining that adding new granular sub-flags is a wildcard
  loosening event.
- Add a test that constructs an `AskForApproval::Granular` with the
  *minimum* required fields and confirms the wildcard accepts it; and
  another with the *maximum* fields including non-required ones.
- Document in the v2 schema changelog that `allowedApprovalPolicies`
  item type changed.

## Severity

Medium. Approval-policy allow-lists are governance-critical surfaces;
a wildcard's semantics (and the rules for how new granular flags
interact with existing wildcards) need to be explicit, not implicit.
