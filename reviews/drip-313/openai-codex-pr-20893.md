# openai/codex PR #20893 — Add issue labeler area labels

- Author: etraut-openai
- Head SHA: `a31e6182c8b53c8dcb4d4dd88ffc901158bfd0f8`
- Verdict: **merge-as-is**

## Rationale

Pure workflow tweak to `.github/workflows/issue-labeler.yml`: extends the
prompt instructions at line 47 to steer "memory" / "performance" labels,
and adds 8 new area labels (computer-use, browser, memory, imagen, remote,
performance, automations, pets) at lines 71-78 with the fallback "agent"
guidance at line 79 updated to enumerate the new ones. The numbering is
contiguous (24 → 32) and the fallback list correctly mirrors the new
additions. No code paths affected — labeler runs on issue events only.
Ship.
