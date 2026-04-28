# BerriAI/litellm #26680 — Add MseeP.ai badge

- PR: https://github.com/BerriAI/litellm/pull/26680
- Author: mseep-ai (MseeP.ai bot)
- Head SHA: `1f8d31f5d68b`
- Base: `litellm_internal_staging`
- Diff: 2+/0- across 1 file (`README.md`)

## Verdict: request-changes

## Rationale

- **This is a vendor-solicited badge addition from a third-party "security assessment" service that the project did not request.** The PR body (sent by the badge vendor's own bot account `mseep-ai`) explicitly frames it as marketing: "we have an entry for litellm in our directory ... we invite you to add our badge ... visit it at https://mseep.ai/app/berriai-litellm." The badge image at `README.md:1` (`https://mseep.net/pr/berriai-litellm-badge.png`) and the link target at the same line (`https://mseep.ai/app/berriai-litellm`) both point to the vendor's own infrastructure — i.e., adding this badge gives `mseep.net` a referer-leak / pixel-tracking surface on every README impression, plus the vendor controls what the badge image *says* over time (today: "Security Assessment Badge"; next quarter: anything they want).
- **The "security assessment" itself appears to be vendor-side directory listing, not a third-party audit relationship the project signed up for.** The PR body doesn't reference a SOC 2, OWASP review, code-audit deliverable, or any artifact the project maintainer would have approved. Embedding a "Security Assessment Badge" implies an endorsement relationship to readers (especially security-conscious enterprise users evaluating LiteLLM as a proxy in their stack) that may not exist.
- **The base branch is `litellm_internal_staging`, which is the canonical landing branch for vendor-team registry adds and similar batched changes** — but a README-top badge is *not* a registry/vendor-config change; it's a top-of-document marketing element that should land on a more visible review surface (probably `main` direct, with explicit maintainer approval). Routing through the staging branch is suspicious shape: it skips the visibility that a `main`-targeted PR would get.
- **Mechanically the diff is correct and minimal** — two lines added at `README.md:1-2` (one img-link Markdown line + one trailing blank line) ahead of the existing `<h1 align="center">` block. There's no risk of breaking anything; this is a pure visibility/policy decision, not a correctness one. But the right verdict for a maintainer-scope decision masquerading as a trivial doc PR is "request changes (close, or escalate to maintainer)."
- **Recommendation**: maintainers should close as decline-by-policy unless there's an explicit prior agreement with MseeP.ai that the project signed up for. If the project *does* want a third-party security-attestation badge, it should solicit one (Snyk, OSSF Scorecard, FOSSA, etc.) on its own terms with a known scope of attestation, and choose the placement deliberately. A vendor-bot-submitted top-of-README badge is the wrong shape regardless of the vendor's intent.

## Nits / follow-ups

- If accepted (which I'd recommend against), at minimum: (a) move the badge below the existing `<h1>` rather than above it so the project name is the first line a reader sees, (b) confirm with maintainers that `mseep.net` is on the project's allow-list of third-party image hosts (some projects pin all README images to GitHub-hosted assets via `user-attachments` to avoid third-party tracking pixels and content-drift), (c) ask MseeP.ai to commit to badge-content stability via a versioned URL rather than the plain `/pr/berriai-litellm-badge.png` shape that lets the vendor change what the badge *says* without the project's involvement.
- The PR body itself is a marketing pitch with a Markdown horizontal rule separating the marketing text from the "the patch you'd be applying" block — that's also a flag for "this came from a templated outreach campaign, not a contributor who uses the project."

## What I learned

Vendor-solicited badge PRs are a recurring class of low-value-but-attractive-looking contribution that maintainers should have a default-decline policy for unless the badge represents a relationship the project initiated. The "trivial diff (2 lines, no code)" shape is exactly what makes them tempting to merge without thinking — the cost looks like zero, but the long-tail cost (referer-leak on every README impression, vendor-controlled badge content drift, implicit endorsement of an unverified attestation) is real. The "base branch is `_internal_staging`" routing is also a teaching moment: review checklists should flag any cross-repo / external contribution that targets a non-`main` branch, since the choice often signals "trying to land with less visibility than usual." This applies symmetrically to the analyzer-side of any review-bot pipeline: vendor-bot-submitted PRs from accounts that match the badge URL host are a strong signal to escalate to a human, not auto-approve.
