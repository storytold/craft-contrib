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
[BridgeCraft](https://github.com/adikshan11/bridgecraft) ("Studio",
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
`https://bridgecraft.onrender.com/studio`:

- **Sign-in**: OAuth with any OIDC provider that issues JWT access tokens
  (Auth0 today). Each user gets an opaque id. Their files live under
  `users/<id>/` in S3-compatible storage, with upload and download links that
  expire. Files attached in ChatGPT, on a phone too, come in through
  `openai/fileParams`, so people don't need a public link.
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
  `claude plugin install bridgecraft --marketplace adikshan11/bridgecraft`),
  an "Add to Cursor" link and a VS Code `code --add-mcp` command. Each points
  at the same `/studio` address, and a test keeps them in step.
- **Checks, and what they are not.** BridgeCraft's CI
  (`.github/workflows/ci.yml`) runs the unit tests, builds the image, and
  runs `tests.test_craft_real` inside that image. That file checks
  `/tmp/escape.*` and `../escape.*` on the apps the image has installed. The
  fake CLI rejects a resolved path outside its root; its comment says that
  includes symlinks. The image runs as a non-root user and has a health
  check, and the Actions in that workflow are pinned by commit. The
  repository does not contain pip-audit, trivy, bandit, semgrep, gitleaks,
  hadolint, checkov or zizmor, and it has no load test, no downtime drill
  and no penetration test. `/status` shows
  how long this process has been up; the code says that figure resets on
  restart and is not a stored metric. craft-contrib's only workflow is the
  shared SonarCloud wrapper. It skips instrumented tests when the repository
  has no Rust crate, and it skips the scan step on a fork pull request when
  the token is absent. A green check from that wrapper is not a security
  proof. These are the pre-beta checks. Storytold's security review (below)
  is still required before hosting.

The community process deletes a file 7 days after it was written, on an
hourly sweep (`STUDIO_FILE_RETENTION_DAYS`, default 7). The hosted rule in
the proposal is 7 days after the session ends.

## Proposal

1. **Move the repository** to `storytold/craft-connector` (name open). The
   current author stays on as a maintainer of the connector code and keeps
   fixing it. Storytold reviews the code before the repository moves and
   before the service runs under the org's name. The author remaining a
   maintainer does not replace that review, and that review does not replace
   the security review in item 3.
2. **Run it at** `https://connect.getartcraft.com/mcp`. That address is the
   connector, not an example domain. Storytold operates the instance, the
   network and the storage, and pays for them. On-call is a Storytold
   rotation. The author maintains the connector code and is the first
   responder, and is not the only person on call. Once the service runs under
   the org, the user files and the tokens are Storytold's responsibility.
   The community repository staying Apache-2.0 does not make that hosted
   data a public archive. PII and file contents stay inside `users/<id>/`.
   Logs must not contain file contents, tokens, or raw paths outside the
   connector.
3. **Sign in with ArtCraft as two audiences** once `artcraft-services`
   offers OAuth to third-party apps. The audience of the connector token is
   the connector's own URL, `https://connect.getartcraft.com/mcp`. The
   connector rejects a token whose audience is anything else. It checks the
   issuer, the signature, the expiry, the audience and the scopes, and it
   does not forward that token. Generation and library reads use a second
   token from an exchange, scoped to those two only. That exchanged token is
   stored encrypted, only on the connector, only while the grant lasts. It
   is not written to logs, not put in the client, and not forwarded.
   Disconnect, account close, or revocation at ArtCraft deletes it. Each
   paid job still asks before it spends credits. Before the service runs
   under Storytold's name or inside ArtCraft beta, Storytold does a security
   review of the scopes, where that token is stored, and revocation. The
   code review in item 1 does not replace this review.
4. **Keep the listings in one place.** The author publishes BridgeCraft to the
   directories, once, for everyone: Anthropic's directory (claude.ai,
   the desktop app, Cowork and Claude Code), the ChatGPT apps directory, and
   the official MCP Registry. People then add it to their own account from
   the listing, or from getartcraft.com, with nothing to set up. If Storytold
   adopts the connector, the same listings move to Storytold's name and
   ArtCraft sign-in; until then, getartcraft.com only needs to link to them.
5. **Admit an app** to the connector only when its latest release passes
   RFC-0015's escape tests. Its manifest entry is updated in the same PR that
   bumps the version. CADCraft, FilmCraft, GridCraft, LightCraft and
   EffectCraft stay off the connector until that is true. Apps still waiting
   on a folder lock stay off.
6. **Publish Storytold's own privacy statement, terms and support pages.**
   Files stay under `users/<id>/` in storage Storytold controls. In beta,
   files under `users/<id>/` are deleted 7 days after the session ends, and
   immediately when the person deletes them, disconnects, or the ArtCraft
   account is closed. A person can delete their files without closing the
   account. The privacy statement Storytold publishes states that 7-day
   period.

## What it needs from other repos

| Repo | Change |
|---|---|
| `artcraft-services` | OAuth for third-party apps ("Sign in with ArtCraft"). The connector audience is `https://connect.getartcraft.com/mcp`. A separate exchanged token, scoped to generation and library read, calls the API (#2676). The connector stores that token encrypted until disconnect, account close, or revocation. |
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
