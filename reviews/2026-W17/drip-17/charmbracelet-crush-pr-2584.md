# charmbracelet/crush PR #2584 — feat(agent): allow user to configure agent model size

- **Author:** BrunoKrugel
- **Head SHA:** 72b78a9b37dd53c74f1c4aaec9bf31f43787c979
- **Files:** `internal/config/config.go`, `internal/config/load_test.go`,
  `schema.json` (+127 / −3)
- **Verdict:** `request-changes`

## Context

Crush has two built-in agents: `coder` (write/edit) and `task`
(read-only context search). Both were hard-coded to
`SelectedModelTypeLarge`. Many users want `task` to run on the
small/cheap model since it's just searching, while keeping `coder` on
large. This PR threads a per-agent `model` field from the config
JSON into `SetupAgents`.

## What the diff does

- `internal/config/config.go:395`: the `Agents` field flips from
  `json:"-"` to `json:"agents,omitempty"` with a jsonschema tag.
  Now agents are user-loadable from config.
- `internal/config/config.go:518-525`: `SetupAgents` reads
  `c.Agents[AgentCoder].Model` / `c.Agents[AgentTask].Model` and
  falls back to `SelectedModelTypeLarge` when missing or empty.
- `schema.json` lines 6–55: new `Agent` definition exposing `id`,
  `name`, `description`, `disabled`, `model` (enum: `large` |
  `small`), `allowed_tools`, `allowed_mcp`, `context_paths`. The
  `agents` field is added to the top-level Config schema.
- Tests at `load_test.go:24-39` and `:516-555` cover JSON load and
  setup.

## Review notes

- **Blocking — schema lies about required.** `schema.json` lines
  53–55 declare `"required": ["model"]` on `Agent`, but the test at
  `load_test.go:24-39` loads `{"agents": {"task": {"model": "small"}, "coder": {"model": "large"}}}`
  and works. The runtime code at
  `config.go:519` happily handles `agentCfg.Model == ""` by
  falling back to `SelectedModelTypeLarge`. So the schema marks
  `model` as required, but the code treats it as optional. Pick
  one. Given the whole point of this PR is "let me override one
  agent without configuring everything," the schema should drop
  `model` from `required`.
- **Blocking — schema exposes fields the user can't set.** The new
  `Agent` schema lists `id`, `name`, `description`, `disabled`,
  `allowed_tools`, `allowed_mcp`, `context_paths`. None of these
  are wired through `SetupAgents` in this PR — the function
  unconditionally rebuilds `agents := map[string]Agent{...}` at
  `config.go:526` with hard-coded ID/Name/Description and
  *re-derives* AllowedTools/ContextPaths from `c.Options`. So
  user-provided `name`/`description`/etc. are silently dropped.
  Either trim the schema to `{model}` only, or actually merge those
  fields in `SetupAgents`.
- **Inconsistency — `disabled` field.** Schema has it, code never
  reads it. If a user writes `{"agents": {"task": {"disabled": true}}}`
  the task agent still gets created. Surprise behaviour.
- **Naming.** `model` of type enum `large|small` is fine but
  collides with the user's mental model — they'll expect to put
  `gpt-4.1` or `claude-sonnet-4-6` here. A doc comment in the
  schema (`"description": "Logical model size to bind: 'large' or 'small'. Maps to the global model selection — see Models config."`)
  would prevent the inevitable issue thread.
- **Test coverage gap.** `TestConfig_setupAgentsWithModelConfig`
  only covers task=small + coder=default. Add: both agents
  configured, both unset, task with empty `""` model (the
  `agentCfg.Model != ""` guard).
- The fall-back logic at `config.go:519-524` is correctly
  defensive — `c.Agents` may be nil from older configs.

Net: the runtime change is small and right; the schema is the
problem. Once schema/code agree on what's user-settable, this is
mergeable.

## What I learned

When you flip a field from `json:"-"` to user-loadable, the schema
becomes a contract, and any field you expose without wiring is a
silent footgun. It's cheaper to ship a tiny schema with one field
(`model`) and grow it than to publish six fields and have five of
them be no-ops.
