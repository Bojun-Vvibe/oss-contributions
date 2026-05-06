# QwenLM/qwen-code#3864 — feat(cli): refactor auth around provider registry

- Head SHA: `12a867f042020c392a78bb8007b74bcff838b995`
- Author: @pomelo-nwu
- Link: https://github.com/QwenLM/qwen-code/pull/3864

## Notes
- `docs/design/auth/motivation.md:1–132` (new) is a thoughtful design doc proposing four user-facing entries (Alibaba ModelStudio / third-party / OAuth / custom) unified internally as `provider + setup method + install plan + source`. The proposed `packages/cli/src/auth/` tree (registry, install, providers/{alibaba,thirdParty,oauth,custom}) is clean and discoverable.
- The doc is in mixed Chinese/English — fine for an internal design note, but consider an English summary at the top so non-CN reviewers can ack the architectural direction without translation.
- Risk: this is a structural refactor of auth — historically a high-blast-radius area. The PR description should enumerate which existing settings.json shapes are preserved bit-for-bit and which are migrated. Diff excerpt only shows the design doc; need to see the actual code moves before approving.

## Verdict
`needs-discussion`

Direction is sound and the proposed tree matches the four-entry UX cleanly. Hold for: (1) explicit migration matrix for existing `~/.qwen/settings.json` shapes, (2) English summary of the design intent, (3) confirmation that OAuth provider list (modelscope/openrouter/fireworks) is final before code freeze.
