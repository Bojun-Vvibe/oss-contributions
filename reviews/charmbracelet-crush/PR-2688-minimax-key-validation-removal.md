# PR #2688 — remove MiniMax API key prefix validation

**Repo:** charmbracelet/crush • **Author:** flynn-eye • **Status:**
merged • **Net:** +0 / −4

## Change

In `internal/config/config.go`, the `TestConnection` switch arm for
`InferenceProviderMiniMax` and `InferenceProviderMiniMaxChina` drops
the `if !strings.HasPrefix(apiKey, "sk-")` check. The comment above
("MiniMax has no good endpoint we can use to validate the API key.
Let's at least check the pattern.") becomes obsolete; only the first
sentence is kept.

## Why this is correct

MiniMax issues two distinct flavors of API keys:

1. Standard API keys with the `sk-` prefix (the historical default).
2. **Token plan** keys, which use a different opaque format
   (typically a long base64-ish blob without the `sk-` prefix).

The original validation assumed (1) was the only shape. Users
buying token plans got "invalid API key format" at startup, and the
provider could not be configured at all. Since the validation was a
**format check**, not an actual auth check, removing it doesn't
weaken security — it just stops rejecting keys that the upstream
API would have accepted.

## What's lost (and why it doesn't matter much)

The `sk-` prefix check was the only client-side guard against:

- A user pasting in a partial key (e.g. `sk` cut off mid-paste).
- A user pasting in a placeholder like `your-api-key-here`.

Both classes of mistake will now fail at first model call instead
of at config load. That's a worse UX in a narrow sense — error
deferred from minutes-to-seconds — but the error message at the
call site (HTTP 401 from MiniMax) is more honest than a fabricated
"invalid format" lie that can't tell the user *why* the format is
invalid.

A defensive alternative would have been to split the check: accept
either `sk-…` or a length+charset heuristic for token-plan keys.
That's brittle (the token-plan format is undocumented and may
change), and the team correctly chose "accept anything, let the
server decide" — which matches what every other provider arm in
this same `TestConnection` switch does. The MiniMax check was the
outlier; removing it is consistency, not regression.

## Sharp edge

The comment is now misleading. It still says "MiniMax has no good
endpoint we can use to validate the API key" — which is the
*reason* the check existed. After this PR, the comment should
either be deleted or rewritten to explain that no client-side
validation is performed because the key formats are not stable
across token plan tiers. Otherwise a future contributor reads
"MiniMax has no good endpoint" and thinks "I should add a pattern
check" — exactly the regression this PR is preventing.

## Concrete next step

Replace the comment with: `// MiniMax accepts multiple key formats
(sk-prefixed standard keys and unprefixed token-plan keys); skip
client-side validation and let the upstream API authoritatively
reject bad keys.` That documents the *decision*, not the absence of
an endpoint.

## Verdict

Correct fix for a real "the validator is wrong" bug. The trade-off
(lose paste-mistake catch, gain token-plan support) is the right
one — token-plan support is a real use case, paste-mistake catch is
a UX nicety that can be reintroduced as a length-based heuristic
later if support volume warrants it. The leftover misleading
comment is the only remaining nit.

## What I learned

Pattern-based key validation is almost always wrong for SaaS
providers, because providers add new key tiers (free, paid, token
plan, enterprise) on their own schedule and rarely commit to a
prefix contract. The right validation surface is "make a real auth
call against a cheap endpoint"; if no such endpoint exists, the
right answer is to skip validation, not to invent a fake one.
