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

## What the connector does

Studio is a Python MCP server (streamable HTTP). The community deployment is
live at `https://bridgecraft.onrender.com/studio`. The controls below are what
that code does on the request path.

- **Sign-in.** Auth0 is the sign-in service. The community deployment's
  authorization server is `https://artcraft-studio.us.auth0.com/`. The
  connector accepts a token only when its audience is exactly
  `https://connect.getartcraft.com/mcp`. A token whose audience is anything
  else is rejected, including a list of audiences, an extra audience beside
  that URL, or a trailing slash. Studio checks the issuer, the signature
  (RS256, ES256, or PS256 against the issuer's JWKS), the expiry, the
  audience, and the scopes. The required scope is `connector`; extra scopes
  are allowed. The issuer is `OAUTH_ISSUER`. `OAUTH_AUDIENCE` is not read, so
  it cannot widen what is accepted. Each user gets an opaque id. Files
  attached in ChatGPT, including on a phone, come in through
  `openai/fileParams`.
- **Where the community deployment runs.** Auth0 handles sign-in. File
  storage is a private Backblaze B2 bucket when the `S3_*` variables are set;
  Cloudflare R2 is the other storage path, used only when the `R2_*` variables
  are set instead. Render runs the process (service `bridgecraft` at
  `bridgecraft.onrender.com`, health check at `/health`). The image runs as
  the non-root user `studio`. Generation and library reads call
  `https://api.storyteller.ai` with the exchanged token. The public project
  catalog calls `api.github.com`, and that call refuses the connector token
  as well. Upload and download links expire (15 minutes and one hour). The
  process answering that host on 1 November 2026 still publishes
  `https://bridgecraft.onrender.com/studio` as its protected resource. This
  tree publishes and accepts only `https://connect.getartcraft.com/mcp`.
- **The connector token stays on the connector.** It is not sent to ArtCraft
  or to any other upstream API. Listing models and estimating a cost send no
  `Authorization` header. Generation, job status, and library reads use the
  exchanged token below. The connector token is presented only as the RFC 8693
  subject token, and only to the issuer's token endpoint, over HTTPS, on the
  issuer's own host. If `OAUTH_TOKEN_URL` names any other host, the exchange
  is refused before it is sent. A call that would attach the connector token
  to ArtCraft is refused before the connection opens. The public GitHub
  catalogue call refuses it the same way.
- **Generation and library reads.** Those two uses exchange the connector
  token for a second token. The exchange asks for audience
  `https://api.storyteller.ai` and for scopes `generation` and `library:read`
  and nothing else. The returned token must be a different string, its scopes
  must be exactly those two, and `expires_in` must be an integer from 1
  second through 30 days. Studio stores it encrypted (Fernet) at
  `users/<id>/grant/exchanged-token`, only on the connector, only while the
  grant lasts. On Render, a missing or invalid `GRANT_KEY` means the grant is
  not stored. Off Render, the process uses an ephemeral key that is never
  written down. The token is not logged, not returned to the assistant, and
  not written into the repository. Disconnect, account close, and revocation
  delete it. Revocation is ArtCraft answering 401 on a call made with the
  exchanged token: Studio deletes the grant, marks the session ended, and
  closes that user's Craft sessions. Stored files then follow the 7-day
  clock. `disconnect_account`, `close_account`, `POST /account/disconnect`,
  and `POST /account/close` delete the grant and every object under
  `users/<id>/` immediately. A paid generation still starts only after the
  caller passes `max_credits` that covers the price.
- **Files.** User data stays under `users/<id>/`. Files there are deleted 7
  days after the session ends, and immediately when the person deletes that
  file, disconnects, or the account is closed. The session ends after
  `CRAFT_IDLE_MINUTES` (default 10) without a request. An active session keeps
  its files, however old those files are. The sweep runs
  at startup and every hour. `STUDIO_FILE_RETENTION_DAYS` is 7; any other
  value is ignored. The deploy notes say a bucket lifecycle rule of that age
  is a backstop for media objects. Usage counts live under `ops/` and hold no
  paths, inputs, file contents, or account ids. The session clock is
  `users/<id>/session/session.json` and stores `last_active` and `ended_at`
  only.
- **Logs.** Tools do not log file contents. A logging handler redacts JWTs,
  bearer values, ArtCraft job tokens, and the process secrets `GRANT_KEY`,
  the S3 and R2 keys, `ARTCRAFT_API_KEY`, `GITHUB_TOKEN`, and
  `AWS_SECRET_ACCESS_KEY` when that value is at least 8 characters. Exception
  text is redacted the same way.
- **Limits the code enforces.** 500 MB per file (`STUDIO_MAX_FILE_MB`, default
  500) and 5000 MB per account (`STUDIO_USER_QUOTA_MB`, default 5000). Each
  user may have 2 Craft sessions open, and the server 6
  (`CRAFT_MAX_SESSIONS_PER_USER`, `CRAFT_MAX_SESSIONS`). A session goes idle
  after 10 minutes. A Craft call is limited to 120 seconds and 1536 MB by
  default. ArtCraft calls are counted in memory, per user, per hour: 30
  generations, 120 library reads, and 120 job checks. The terms forbid
  unlawful uploads, malware, and reaching other users' files, and say an
  account may be suspended. The hosted connector does not attach to a desktop
  app. Desktop sessions remain #1917.
- **Library call.** `list_artcraft_library` calls
  `GET /v1/media_files/project/list` on `https://api.storyteller.ai` with the
  exchanged token. It returns `media_token`, title, project type, media
  class, and `created_at` (at most 50). `media_token` there is an ArtCraft
  media id, not the auth token.
- **Tests.** `tests/test_connector_auth.py` rejects a token for any other
  audience, a bad signature, issuer, expiry, or scope, and an
  `OAUTH_AUDIENCE` that names a different URL. It refuses to send the
  connector token to ArtCraft, GitHub, or storage. The generation tool sends
  the exchanged token, and that token is absent from the tool result, the
  stored grant, and the log. Files stay until 7 days after the session ends,
  and disconnect, a single-file delete, and `POST /account/disconnect` remove
  them immediately. A wrong audience on that route is rejected and the files
  stay. Revocation deletes the grant and leaves the files for the 7-day
  clock. A file's contents are not written to the log. `python -m unittest
  discover -s tests` on this tree passed: 153 tests, 8 skipped.

Craft apps run as one `<app>-cli mcp` process per user session, in a private
temporary folder, with `--automation-read-root` and `--automation-write-root`
set to that folder (RFC-0015), a minimal environment, and memory, size, time,
and idle limits. `craft_apps.json` pins each app's release by sha256 and lists
the tools it hides and the engine commands it refuses. WordCraft, DeckCraft,
PdfCraft, PhotoCraft, VectorCraft, DesignCraft, FigCraft, and SoundCraft are
hosted. CADCraft, FilmCraft, GridCraft, and LightCraft stay off until their
folder lock lands, and EffectCraft stays off until effectcraft#2719. Everyday
edits without a Craft app use FFmpeg, Pillow, and the PDF tools.

Directory pages (privacy, terms, support), tool annotations, and tool names of
64 characters or fewer are in the repo. `server.json` is the MCP Registry
entry (`io.github.adikshan11/bridgecraft`). A Claude Code plugin is ready for
Anthropic's directory (`claude plugin install bridgecraft --marketplace
adikshan11/bridgecraft`). An "Add to Cursor" link and a VS Code
`code --add-mcp` command point at the same `/studio` address, and a test
keeps them in step.

BridgeCraft's CI (`.github/workflows/ci.yml`) runs the unit tests, builds the
image, and runs `tests.test_craft_real` inside that image. That file checks
`/tmp/escape.*` and `../escape.*` on the apps the image has installed. The
image runs as a non-root user and has a health check, and the Actions in that
workflow are pinned by commit. The repository does not contain pip-audit,
trivy, bandit, semgrep, gitleaks, hadolint, checkov, or zizmor, and it has no
load test, no downtime drill, and no penetration test. `/status` shows how
long this process has been up; that figure resets on restart and is not a
stored metric. craft-contrib's only workflow is the shared SonarCloud
wrapper. It skips instrumented tests when the repository has no Rust crate,
and it skips the scan step on a fork pull request when the token is absent.
A green check from that wrapper is not a security proof.

The community privacy page says the site is not operated by Storytold. It
also states the audience, the encrypted grant, deletion 7 days after the
session ends, immediate deletion on delete, disconnect, or account close, and
that logs do not contain file contents, tokens, or secrets.

## Proposal

1. **Move the repository** to `storytold/craft-connector`. The author maintains
   the connector code. Once the service is hosted, Storytold operates the
   instance, the network, and the storage. User files and the exchanged token
   stay under `users/<id>/`. The Apache-2.0 license on the source does not
   publish that hosted data.
2. **Run it at** `https://connect.getartcraft.com/mcp`. That address is the
   connector, the same string the token audience uses. Storytold operates the
   instance, the network, and the storage. The author maintains the connector
   code.
3. **Sign in as described above.** `artcraft-services` issues the connector
   token for `https://connect.getartcraft.com/mcp` and accepts the exchanged
   token on generation and on `GET /v1/media_files/project/list` (#2676). The
   connector already makes that library call with the exchanged token only.
4. **Keep the listings in one place.** The author publishes BridgeCraft to the
   directories, once, for everyone: Anthropic's directory (claude.ai, the
   desktop app, Cowork and Claude Code), the ChatGPT apps directory, and the
   official MCP Registry. People then add it to their own account from the
   listing, or from getartcraft.com, with nothing to set up. If Storytold
   adopts the connector, the same listings move to Storytold's name and
   ArtCraft sign-in; until then, getartcraft.com only needs to link to them.
5. **Admit an app** only when its latest release passes RFC-0015's escape
   tests, with the manifest updated in the same change as the version.
   CADCraft, FilmCraft, GridCraft, LightCraft, and EffectCraft stay off until
   that is true.
6. **Publish the privacy statement, terms, and support pages under Storytold's
   name** once Storytold hosts the service. The pages in the repository
   already state the 7-day period, immediate deletion, the audience, where the
   exchanged token is stored, and the log rule. On the community site those
   pages say the site is not operated by Storytold.

## What it needs from other repos

| Repo | Change |
|---|---|
| `artcraft-services` | Issue connector tokens for audience `https://connect.getartcraft.com/mcp`, with the `connector` scope, and accept the exchanged token (scopes `generation` and `library:read`) on generation and on `GET /v1/media_files/project/list` (#2676). The connector already calls that route with the exchanged token and does not send the connector token. |
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
  app on their own computer. It doesn't reach the web or phones. The hosted
  connector does not bridge to a desktop app.

## Decision process

A Storytold maintainer comments the RFC-0000 comment-period end date on this
pull request. This RFC does not set that date.
