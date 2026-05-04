# google-gemini/gemini-cli#26454 — feat(voice): add privacy and compliance UX warning for Gemini Live backend

- PR ref: `google-gemini/gemini-cli#26454` (https://github.com/google-gemini/gemini-cli/pull/26454)
- Head SHA: `61342291b2342958e3ea8dee194b6b45965e8221`
- Title: feat(voice): add privacy and compliance UX warning for Gemini Live backend
- Verdict: **merge-after-nits**

## Review

The substance of the change is good and overdue. Voice transcription that ships
audio to a cloud endpoint is exactly the sort of thing enterprise users need an
unmistakable in-product signal for — relying on docs alone is insufficient.

The implementation in `packages/cli/src/ui/components/VoiceModelDialog.tsx`
threads a new `highlightedBackend` state through the existing
`DescriptiveRadioButtonSelect` via `onHighlight={handleBackendHighlight}` (lines
~74-78 and ~108-110 of the new file), and renders a `WarningMessage` only when
`highlightedBackend === 'gemini-live'` (lines ~221-228). That's the right
trigger surface — the warning appears the moment the user moves to the option,
not only after they confirm, which is what makes it actually compliance-useful.

The settings-schema description update in `settingsSchema.ts` propagates
correctly through `docs/cli/settings.md`, `docs/reference/configuration.md`,
and `schemas/settings.schema.json` (visible in the diff at lines ~9-10 and
~26-27), so machine-readable consumers of the schema get the same notice.

Nits:

1. The warning text at `VoiceModelDialog.tsx` ~line 224 says "voice recordings
   are sent to Google Cloud for transcription." That's accurate for the audio
   stream itself, but enterprise compliance reviewers will also want to know
   whether transcripts are *retained* server-side and for how long. Either
   extend the message with a one-line retention statement or link to a stable
   docs anchor — otherwise the warning answers half the compliance question.
2. The new `VoiceModelDialog.test.tsx` covers the conditional render of the
   warning but doesn't assert it disappears when the user moves the highlight
   back to `'whisper'`. The on/off toggle is the user-visible behavior;
   worth one explicit assertion of the off-direction transition.
3. Updating `SettingsDialog.test.tsx` snapshots is fine, but worth a
   one-sentence note in the PR body that the snapshot delta is purely the
   description-string change so a reviewer doesn't have to diff snapshots
   to confirm.
