# RFC-0016: A hosted connector for the Craft apps

Related: RFC-0015 (confined file access), storytold/artcraft#2003 (agent
tools), storytold/artcraft#1917 (MCP sessions), storytold/artcraft#2676 (API
key access).

## Summary

People should be able to add the Craft apps to ChatGPT, Claude or another
assistant from its connector directory, sign in with their ArtCraft account,
and then edit their own photos, documents, drawings and audio by asking for
it, on the web or a phone, with nothing to install.

A working version of this exists as a community project,
[artcraft-plugin](https://github.com/adikshan11/artcraft-plugin) ("Studio",
Apache-2.0). This RFC proposes that Storytold adopt it: move the repository
into the `storytold` org, run it under a Storytold domain, and list it in the
assistant directories under Storytold's name.

## Motivation

- **The desktop MCP servers need a computer.** `<app>-cli mcp` and ArtCraft's
  MCP sessions (#1917) work for someone with a Mac or PC and a client that can
  start local servers. Most assistant users are on a website or a phone, where
  only remote connectors work.
- **Only Storytold can list it properly.** The ChatGPT and Claude directories
  list a connector under its publisher. A community deployment can't present
  itself as the Craft apps, and people looking for ArtCraft won't trust or
  find a connector that isn't from Storytold.
- **Only Storytold can connect ArtCraft accounts.** Generation spends the
  user's ArtCraft credits. Today that works only with a personal API key on
  the user's own computer. A hosted connector needs people to sign in with
  ArtCraft (OAuth) so their requests use their own credits. That sign-in has
  to come from `artcraft-services`.

## What exists today

Studio is a Python MCP server (streamable HTTP), live at
`https://artcraft-storytold-mcp.onrender.com/studio`:

- **Sign-in**: OAuth with any OIDC provider that issues JWT access tokens
  (Auth0 today). Each user gets an opaque id. Their files live under
  `users/<id>/` in S3-compatible storage, with upload and download links that
  expire.
- **Craft apps**: one `<app>-cli mcp` process per user session, started in a
  private temporary folder with `--automation-read-root` and
  `--automation-write-root` set to it, a minimal environment, and memory,
  size, time and idle limits. A manifest (`craft_apps.json`) pins each app's
  release by sha256 and lists the tools it hides (desktop-only, printing,
  keychains) and the engine commands it refuses (collaboration servers,
  publishing, plug-ins).
- **Apps hosted**: WordCraft, DeckCraft, PdfCraft, PhotoCraft, VectorCraft,
  DesignCraft, FigCraft and SoundCraft. Each was escape-tested before it was
  added (RFC-0015, section 5). CADCraft, FilmCraft, GridCraft and LightCraft
  wait for their folder lock, and EffectCraft waits for effectcraft#2719.
- **Everyday edits** without a Craft app: FFmpeg video and audio, Pillow
  images, PDF tools. This lets an assistant do quick jobs without opening a
  session.
- **Directory requirements**: privacy, terms and support pages, tool
  annotations (read-only and destructive hints), tool names of 64 characters
  or fewer.
- **Install routes**: a `server.json` for the official MCP Registry
  (`io.github.adikshan11/bridgecraft`), a Claude Code plugin ready for
  Anthropic's directory (one command until it is listed:
  `claude plugin install bridgecraft --marketplace adikshan11/artcraft-plugin`),
  an "Add to Cursor" link and a VS Code `code --add-mcp` command. Each points
  at the same `/studio` address, and a test keeps them in step.
- **Security**: reviewed with dependency (pip-audit, trivy), static-analysis
  (bandit, semgrep), secret (gitleaks), Dockerfile (hadolint, checkov) and
  workflow (zizmor) scanners; the image runs as a non-root user with a health
  check, and GitHub Actions are pinned by commit. CI runs the tests, including
  the real Craft apps inside the built image.

## Proposal

1. **Move the repository** to `storytold/craft-connector` (name open). The
   current author stays on as a maintainer and keeps fixing things.
2. **Run it under a Storytold domain**, for example
   `https://connect.getartcraft.com/mcp`, with storage and hosting Storytold
   pays for.
3. **Replace the sign-in with ArtCraft accounts** once `artcraft-services`
   offers OAuth to third-party apps. The same token then reaches the
   generation endpoints, so the connector can offer image and video
   generation that spends the user's own credits, asking for consent before
   each paid job.
4. **Keep the listings in one place.** The author publishes BridgeCraft to the
   directories, once, for everyone: Anthropic's directory (claude.ai,
   the desktop app, Cowork and Claude Code), the ChatGPT apps directory, and
   the official MCP Registry. People then add it to their own account from
   the listing, or from getartcraft.com, with nothing to set up. If Storytold
   adopts the connector, the same listings move to Storytold's name and
   ArtCraft sign-in; until then, getartcraft.com only needs to link to them.
5. **Admit an app** to the connector only when its latest release passes
   RFC-0015's escape tests. Its manifest entry is updated in the same PR that
   bumps the version.

## What it needs from other repos

| Repo | Change |
|---|---|
| `artcraft-services` | OAuth for third-party apps ("Sign in with ArtCraft"), with a scope that allows generation and reading the user's library (#2676) |
| `craft-libs` and each app | RFC-0015's shared folder lock and escape tests |
| cadcraft, filmcraft, gridcraft, lightcraft | land the folder lock (open PRs listed in RFC-0015) |
| effectcraft | release effectcraft#2719 |

## Alternatives

- **Leave it as a community project.** It keeps working, but it can't be
  listed as ArtCraft, can't use ArtCraft accounts, and depends on one
  person's hosting.
- **Each app hosts its own connector.** That's thirteen sign-ins, thirteen
  listings and thirteen copies of the session and storage code.
- **Desktop only (#1917).** This is the best fit for people working in the
  app on their own computer. It doesn't reach the web or phones.

## Open questions

- Hosting cost and limits per free account: storage, session minutes,
  concurrent sessions.
- Whether the connector should also bridge to a user's running desktop app
  (#1917) when one is connected, so one listing covers both.
- Abuse handling: rate limits per account, and what the terms say about
  uploaded content.
