# BerriAI/litellm#26690 — [Fix] UI - Allow Proxy Admin Viewer to access /api-keys page

- **PR**: https://github.com/BerriAI/litellm/pull/26690
- **Author**: @kimsehwan96
- **Head SHA**: `56cba1729cbc47b8ed0c543be42e9b1339bc808c`
- **Base**: `main`
- **State**: OPEN
- **Scope**: +0 / -10 across 1 file (UI dashboard)

## Summary

Pure-deletion of an early-return that was hard-blocking the entire `UserDashboard` for any user with `userRole === "Admin Viewer"`. The block at `ui/litellm-dashboard/src/components/user_dashboard.tsx:317-326` rendered an "Access Denied — Ask your proxy admin for access to create keys" full-page screen on every page that reused `UserDashboard`, including `/api-keys`. The fix deletes the block entirely; finer-grained per-feature gating presumably lives further down the component tree (e.g., the "Create Key" button itself can be gated, and the `/api-keys` page can render the read-only key list).

## Diff anchors

- `ui/litellm-dashboard/src/components/user_dashboard.tsx:317-326` (deleted) — the entire block:
  ```tsx
  if (userRole && userRole == "Admin Viewer") {
    const { Title, Paragraph } = Typography;
    return (
      <div>
        <Title level={1}>Access Denied</Title>
        <Paragraph>Ask your proxy admin for access to create keys</Paragraph>
      </div>
    );
  }
  ```
  removed wholesale. The `console.log` debug at `:328` is retained, which is correct (drive-by deletes belong elsewhere).

## What this PR does *not* show, and why that matters

This is the kind of one-file UI deletion whose correctness depends entirely on what gating exists *downstream* of the deleted block. The PR diff alone cannot tell us:

1. **Is the "Create Key" button actually gated?** The block's prose said "ask your proxy admin for access to create keys" — implying the page contains a key-creation surface. If the rest of `UserDashboard` renders a `<CreateKeyButton />` unconditionally, this fix replaces a coarse access-denied screen with a "key creation appears to work but POST returns 403" UX — worse than the original behavior. Reviewer must walk the component tree under `UserDashboard` and confirm every mutation surface (create, edit, delete) has its own `if (userRole === "Admin Viewer") { return null }` or `disabled={...}` gate.
2. **Are there other "Admin Viewer" early-returns elsewhere?** A grep for `"Admin Viewer"` across `ui/litellm-dashboard/src/` should produce zero matches if this was the only block; multiple matches mean the role gating is scattered and the fix is partial.
3. **Does the backend enforce the gate?** Front-end-only role checks are theater. As long as the proxy backend rejects `Admin Viewer` mutations on the API side (which appears to be the case from the PR's framing), the worst-case outcome of this UI change is a poor UX, not a privilege escalation. The PR body should explicitly cite the backend enforcement layer (file:line) so reviewers can confirm this assumption.
4. **`/api-keys` viewer surface for Admin Viewers** — the title says the goal is to let Admin Viewers see the `/api-keys` page. Confirm via a separate UI test or screenshot that the page renders the key list (presumably a read-only table) without 500ing on a missing user_id. The deleted `userRole` check was suppressing a broader `UserDashboard` flow that may have done useful initialization that the Admin Viewer path now skips.

## What I'd push back on (nits)

1. **No test added.** A single Playwright/RTL test that asserts "logged in as Admin Viewer, navigate to `/api-keys`, expect key list to render and 'Create Key' button to be hidden or disabled" would close the entire question above in 30 lines.
2. **PR body should enumerate the gating audit.** A 4-line list naming the backend endpoints that enforce Admin Viewer restrictions (e.g., "POST /key/generate returns 403 for Admin Viewer at `proxy_server.py:N`") and the front-end mutation surfaces that need their own gates would let reviewers approve without re-walking the codebase.
3. **The `console.log("inside user dashboard, selected team", selectedTeam)` and `console.log("All cookies after redirect:", document.cookie)` at `:317-318` are pre-existing debug noise that this PR brushes against.** Worth a follow-up cleanup PR; the cookie-dumping log in particular looks like a debug-leftover that shouldn't ship in production.

## Verdict

**needs-discussion** — the deletion is mechanically correct (the block was clearly over-broad), but a one-file UI deletion that loosens role-based access cannot be approved on the diff alone. Recommend the PR author either (a) extend the diff to add downstream gating + a Playwright cell, or (b) explicitly link to the backend-enforcement layer in the PR body so reviewers can confirm the gate is real and front-end is just unblocking the read path.

Repo coverage: BerriAI/litellm (UI dashboard role gating).
