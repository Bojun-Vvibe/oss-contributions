# PR #8867 — feat: add Ocultar PII Refinery extension

- **Repo**: block/goose
- **PR**: #8867
- **Head SHA**: `e355740d`
- **Author**: Edu963 (Edu)
- **Size**: +206 / -0 across 2 files
- **URL**: https://github.com/block/goose/pull/8867
- **Verdict**: **needs-discussion**

## Summary

Documentation-and-registry-only PR that adds a new third-party MCP
extension entry for "Ocultar PII Refinery" — a self-hosted PII
detection/redaction proxy (`uvx ocultar-goose-mcp`) that pre-processes
text to tokenize PII (names, emails, IBANs, phone numbers,
addresses) before any text reaches an upstream LLM API. The PR adds
no code: it adds (1) a tutorial markdown at
`documentation/docs/mcp/ocultar-pii-mcp.md` (184 lines, with the
`unlisted: true` frontmatter so it doesn't appear in the docs nav
without an explicit link), and (2) a 22-line entry in
`documentation/static/servers.json` registering the extension in the
ecosystem catalog with `endorsed: false`, `is_builtin: false`.

## Specific changes

- `documentation/docs/mcp/ocultar-pii-mcp.md` (new, 184 lines): full
  install tutorial with CLI + Desktop tabs, environment variables
  (`OCULTAR_URL` default `http://localhost:8080`, `OCULTAR_API_KEY`
  optional), example prompt/output flow, and a five-row table
  explaining what the upstream tool detects (Dictionary shield,
  Pattern + entropy, Regex rules, Phone validator, Address
  heuristics).
- `documentation/static/servers.json:561-583` — new entry with
  `id: "ocultar-pii"`, `command: "uvx ocultar-goose-mcp"`,
  `link: "https://github.com/Edu963/ocultar"`,
  `endorsed: false`, `is_builtin: false`, two declared env vars
  (both `required: false`).

## Risks

- **Self-promotion / single-author endorsement shape**: the PR
  author (`Edu963`) is also the linked author of the upstream
  `Edu963/ocultar` repository. The catalog correctly marks
  `endorsed: false` so it's an opt-in user-installable extension,
  not a project-blessed integration. The convention in the existing
  catalog appears to support third-party additions of this shape
  (the file already has 561+ entries), but maintainers should
  verify their unwritten policy on third-party catalog additions
  from the upstream's own author. This is the kind of thing that
  can quickly turn the catalog into a npm-style npm-style spam
  surface if there's no intake review.
- **No verification that the upstream package exists**: the catalog
  entry says `command: "uvx ocultar-goose-mcp"` and the install
  notes say `docker compose -f docker-compose.community.yml up`,
  but there's no automated check in the PR that
  (a) `pip install ocultar-goose-mcp` (and therefore `uvx`) actually
  resolves a published package, or
  (b) the docker-compose file exists at the linked repo root.
  A naive user clicking the desktop installer would hit `uvx`
  failing to resolve if the package isn't actually published, which
  is a worse user experience than not listing it at all.
- **`unlisted: true` semantics**: the tutorial is hidden from the
  docs nav, which seems contradictory with adding a discoverable
  catalog entry. Either (a) intentional ("catalog is the discovery
  surface, tutorial is for users who already know they want it"),
  or (b) a paste-error from copying another tutorial template. The
  PR description doesn't clarify. Worth a maintainer ack.
- **Docker-compose claim is unverified**: the install notes say
  `docker-compose.community.yml` exists in the upstream repo. If
  that file doesn't actually exist, the "fastest way" instructions
  are broken from day one. A maintainer would want to spot-check
  the upstream repo before merging.
- **PII detection accuracy claims**: the tutorial table claims five
  detection layers including "Address heuristics across formats"
  and an "enterprise tier AI scanner". These are unverifiable by
  reviewing this PR alone; a user running this in a serious
  pre-LLM redaction pipeline would care a lot whether the regex
  rules cover (e.g.) UK NI numbers, Irish PPS, US SSN with dashes
  vs without, etc. The catalog `description` field reasonably
  punts to the upstream README — fine — but a user-trust-relevant
  claim like "No raw PII ever leaves your infrastructure" should
  ideally be reviewed against the upstream code (does the proxy
  call any external services? telemetry? error reporting that
  could exfil unredacted text?) before being repeated in
  goose-blessed docs.
- **Banned-string scrub**: the PR text and diff contain no banned
  strings from the local guardrail set; the linked author and
  upstream are unaffiliated.

## Verdict

**needs-discussion**: this is a docs-and-catalog addition with no
code risk to goose itself, but two questions for maintainers:
(1) what's the policy on third-party catalog additions self-submitted
by the upstream author, and (2) is there a basic intake check
expected (uvx package resolves, linked docker-compose exists,
upstream repo has at least minimal README) before catalog entries
land. If the policy is "we accept unendorsed third-party entries
freely as long as they're marked `endorsed: false`" — which the
existing 561-entry catalog suggests — this is fine to merge once
those two questions are answered. Otherwise this should sit until
the maintainers articulate a third-party-intake checklist that this
PR can be measured against.

## What I learned

The "extension catalog" surface in CLI-agent projects is a real
governance question that's easy to ignore until it becomes a spam
surface. `endorsed: false` + `is_builtin: false` is the right
escape valve — it lets the project enable a long tail of
ecosystem extensions without implicitly vouching for them. But the
catalog *itself* still acts as a discovery / install-button
surface, so a malicious or misconfigured `uvx <foo>` can ship
through this path without ever touching the project's main code.
The right intake bar is probably: (1) upstream repo exists and is
reachable, (2) the named install command resolves to a real
published package, (3) the linked authentication/setup flow has
been smoke-tested by the submitter, (4) the description doesn't
make claims that goose's docs implicitly endorse (security,
zero-egress, encryption-at-rest, etc.) without upstream
verification. None of those four are *technically* required by
this PR, but a project-level intake template would make
contributions like this much easier to merge unblocked.
