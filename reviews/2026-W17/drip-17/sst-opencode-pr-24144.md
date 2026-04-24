# sst/opencode PR #24144 — docs: add MDNS workaround for WSL path issues in windows-wsl.mdx

- **Author:** viraj1995
- **Head SHA:** 6df53b41141f63262c0d56552421f61fe1b39ecf
- **Files:** `packages/web/src/content/docs/windows-wsl.mdx` (+25 / −0)
- **Verdict:** `merge-after-nits`

## Context

Desktop ↔ WSL connectivity has a long-standing failure mode where the
desktop client passes UNC paths (`\\wsl.localhost\Debian\...`) into a
server that lives inside WSL but is reached via an IP. Resolving the
working directory then fails on the WSL side because UNC paths are not
meaningful inside the Linux namespace. This docs PR adds an MDNS-based
workaround that sidesteps the problem by addressing the server by
hostname instead.

## What the diff does

In `windows-wsl.mdx` lines 67–93 (the new "Using MDNS (Alternative)"
subsection) the doc:

1. Tells the user to enable WSL2 mirrored networking via
   `%USERPROFILE%\.wslconfig` with `networkingMode=mirrored`.
2. Instructs them to start the server with
   `opencode serve --mdns --mdns-domain opencode.local --hostname 0.0.0.0 --port 4096`.
3. Tells them to connect the desktop app to `http://opencode.local:4096`.
4. Explains *why*: hostname resolution avoids UNC paths surfacing in
   the request.

## Review notes

- The ordering is right: mirrored networking has to come first,
  otherwise mDNS multicast won't traverse the NAT-mode adapter at all
  and step 2 silently fails.
- Worth tightening: step 2 says `--hostname 0.0.0.0` *and* `--mdns`.
  The doc above already covers `--hostname 0.0.0.0` requiring
  `OPENCODE_SERVER_PASSWORD`; the new section should cross-link or
  duplicate that warning, otherwise users following the new path will
  bind a wide-open server with mDNS broadcasting it. That is a
  meaningful security regression for the WSL desktop user, who is
  likely on a corporate LAN.
- Step 3 hard-codes `opencode.local`, but `--mdns-domain` is
  user-supplied. Either drop the `--mdns-domain` flag (and rely on the
  default) or call out that the URL must match.
- The "why" sentence at line 92 is correct and load-bearing; keep it.

Add the password warning + the URL/domain consistency note and this is
mergeable as a docs-only change.

## What I learned

mDNS-as-a-workaround for UNC-path leakage is a clever decoupling:
the bug is that an absolute path string crosses a process boundary
that doesn't share its namespace, and substituting hostname
resolution for IP-with-UNC is enough to keep the path layer agnostic.
Worth noting for any cross-namespace IPC where the client side has
its own filesystem view.
