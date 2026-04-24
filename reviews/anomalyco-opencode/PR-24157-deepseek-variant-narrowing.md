# PR #24157 — fix: deepseek variants

**Repo:** anomalyco/opencode  
**Surface:** `packages/opencode/src/provider/transform.ts`  
**Severity:** Low-Medium (correctness, scope)

## What changed

The variant-eligibility branch in `transform.ts:407-412` previously matched any
model id containing the substring `"deepseek"`. The PR replaces that with an
allow-list of four concrete families:

```
deepseek-chat | deepseek-reasoner | deepseek-r1 | deepseek-v3
```

The intent is to stop applying the deepseek-flavored variant treatment to
sibling models that incidentally carry "deepseek" in their id (e.g. embedding
or vision-tower derivatives, or third-party fine-tunes whose ids happen to
contain the token).

## What's risky / wrong / missing

1. **Forward compatibility regression.** A new model family — say
   `deepseek-v4`, `deepseek-coder`, or a future `deepseek-r2` — will now
   silently *fall out* of the variants branch instead of being picked up by
   the substring match. Given the speed at which this provider ships new
   tags, the allow-list will go stale quietly.

2. **No test coverage.** The PR is +4/-1 with no companion test asserting
   that the four targeted ids hit the branch and a non-targeted id (e.g.
   `deepseek-embed-v1`) does not. The only existing signal is whatever a
   downstream snapshot test catches incidentally.

3. **Asymmetry vs. siblings.** The same branch keeps loose substring matches
   for `minimax`, `glm`, `kimi`, etc. If the motivation is "narrow to known
   chat/reasoning families to avoid accidental matches," that motivation
   applies symmetrically. Tightening only one vendor invites the same bug
   class to recur for the others.

4. **Case handling looks correct** (`id.toLowerCase()` upstream), but the
   allow-list values themselves are lowercased literals and rely on that
   normalization being permanent. A defensive comment would help.

## Suggested fix

- Either keep the loose substring match and explicitly *deny-list* the
  narrow set of ids that broke (more future-proof), or
- Land the allow-list as written but add (a) a regex like
  `^deepseek-(chat|reasoner|r\d+|v\d+)` so future numeric bumps are picked
  up automatically, and (b) a unit test that pins both the positive and
  negative cases.

## Severity

Low-Medium. It's a single-line behavioral narrowing on a hot
provider-routing path. Wrong outcome here is "new deepseek release works
in raw chat but its variants behave subtly differently from the previous
release," which is exactly the class of bug users won't open issues about
— they'll just claim the model "feels worse."

## What I'd ask the author

> Have you confirmed that no downstream variant logic (effort levels,
> reasoning extraction) is needed for any deepseek id outside the four
> you listed? If yes, add a regex with a `\d+` placeholder so the next
> point release doesn't quietly drop out.
