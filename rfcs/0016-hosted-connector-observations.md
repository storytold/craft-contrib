# Dated observations for RFC-0016

These checks are not the decision. The decision and its gates are in
[0016-hosted-connector.md](0016-hosted-connector.md). A date here is the day
the check was made. It is not a promise that the same screen still says that.

## Community deployment, 1 November 2026

Auth0 handles sign-in. File storage is a private Backblaze B2 bucket when
the `S3_*` variables are set. Cloudflare R2 is the other storage path, used
only when the `R2_*` variables are set instead. On 1 November 2026 the
Backblaze card for `bridgecraft-studio-files` shows File Lifecycle as Keep
all versions. That card does not show a 7-day lifecycle rule. Render runs
the process (service `bridgecraft` at `bridgecraft.onrender.com`, health
check at `/health`). The image runs as the non-root user `studio`.
Generation and library reads call `https://api.storyteller.ai` with the
exchanged token. The public project catalog calls `api.github.com`, and that
call refuses the connector token as well. Upload and download links expire
(15 minutes and one hour).

The process answering `https://bridgecraft.onrender.com` on 1 November 2026
publishes protected-resource metadata whose resource is
`https://connect.getartcraft.com/mcp`, whose authorization server is
`https://artcraft-studio.us.auth0.com/`, and whose scope is `connector`.
`connect.getartcraft.com` does not resolve. This note does not show that
Auth0 issues a token for that audience. The connector accepts only that
audience. The tenant's API list on that date has two custom APIs: ArtCraft
Studio, audience `https://artcraft-storytold-mcp.onrender.com/studio`, and
bridgecraft-artcraft, audience `https://bridgecraft.onrender.com/studio`.
It also has the Auth0 Management API. It has no API for
`https://connect.getartcraft.com/mcp` and no API for
`https://api.storyteller.ai`. That list does not show the scopes on the two
custom APIs, and this note does not show a completed token exchange.

The Auth0 tenant name on that check is `artcraft-studio`. That is the name
on the sign-in screen. Changing the screen is done in the Auth0 dashboard.

## Versions on the community bucket, 4 November 2026

`/status.json` reports the real bucket's count of versions that are not a
file's current one as `old_versions`. It read 0 on 4 November 2026. That
reading is not the regression in proposal item 7. Item 7 still requires the
four deletion paths against a real versioned bucket.

## Tests at BridgeCraft `8523497`

`python -m unittest discover -s tests` on BridgeCraft `8523497` passed: 210
tests, 8 skipped. That commit is before adikshan11/bridgecraft#66 moved the
pinned Craft releases. This note does not give a count for a later commit.

## Names, checked against adikshan11/bridgecraft#67

#67 is open and not merged. On that branch the MCP server and its client
name are `bridgecraft`, the signed-PDF stamp says BridgeCraft, and the
homepage's first paragraph says the site is a community project and not a
Storytold or ArtCraft product. The Auth0 tenant is still named
`artcraft-studio`.
