# charmbracelet/crush #2584 — feat(agent): allow user to configure agent model size

- SHA: `72b78a9b37dd53c74f1c4aaec9bf31f43787c979`
- State: OPEN, +127/-3 across 3 files
- Files: `internal/config/config.go` (+12/-3), `internal/config/load_test.go` (+57), `schema.json` (+58)
- Closes: #2501, #427 — Related: #914, #2555, #2365, #2364, #2616

## Summary

Allows users to opt the Task agent (and Coder agent) into the `small` model via `config.json` `agents.task.model` / `agents.coder.model`. Removes the `json:"-"` tag on `Config.Agents` so the field is actually unmarshalled, then in `SetupAgents()` looks up a per-agent override before falling back to `SelectedModelTypeLarge`. Defaults are unchanged.

## Notes

- `internal/config/config.go:395` — flipping `json:"-"` to `json:"agents,omitempty"` is the core enabler. This is a soft schema change — any existing `config.json` with a stray `"agents": ...` key (previously silently ignored) will now actually take effect. Worth a release-note callout.
- `internal/config/config.go:518-525` — coder/task model lookup is symmetric and falls through to `SelectedModelTypeLarge`. Clean. Consider extracting a helper `func (c *Config) modelFor(agentID string) ModelType` to dedupe and make future agents (subagent, etc.) trivial to add.
- `internal/config/config.go:535,541` — only `Model` is honored from a user-supplied `Agent` block. Other fields on the user-supplied `Agent` (Name, Description, AllowedTools, ContextPaths, AllowedMCP) are silently overwritten by the SetupAgents defaults. That's the safe default but also a footgun — a user setting `agents.coder.allowed_tools` will see no effect and no error. Either honor more fields or log a warning when unknown sub-fields are present.
- `schema.json:6-58` — generated `Agent` definition lists `model` as `required`, but `SetupAgents()` tolerates an empty `agents.task` block (treats missing `Model` as the large default via the `agentCfg.Model != ""` guard). Schema and behavior disagree: schema says required, code says optional. Either drop `required: ["model"]` from the schema or make the empty-string case validation-rejected.
- `load_test.go:24-37` — `TestConfig_LoadAgentsFromJSON` covers the unmarshal path. Good.
- `load_test.go:516-535` — `TestConfig_setupAgentsWithModelConfig` verifies override-task-keep-coder-default. Good.
- `load_test.go:540-555` — `TestConfig_setupAgentsWithNoModelConfig` covers the both-default case. Good.
- Missing test: invalid `model` value (e.g., `"medium"`). The schema enum lists only `large`/`small`; the Go code does no validation. A bad value would silently become an empty `SelectedModelType` — would be good to either reject or coerce to the default with a warning.

## Verdict

`merge-after-nits` — feature is small, well-tested for the happy paths, and matches user demand. Address: (1) schema/code disagreement on `model` required-ness, (2) silent ignore of other agent-block fields, (3) one negative-path test for invalid `model` value.
