# Licences

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as in RFC 2119 and RFC 8174.

These rules apply to every dependency, asset and piece of copied code in the Craft repositories. `AGENTS.md` carries the short form and links here. Adding an identifier needs an RFC, and only when a dependency or asset uses it. Lines that state a licence's own conditions use MUST.

## Our own licence

Our code is `MIT OR Apache-2.0`. A new crate and a new repository say so (the crate's `license` field, `LICENSE-MIT` and `LICENSE-APACHE`). Licences are set per crate; files need no SPDX header.

## The allowlist

Dependencies and assets use SPDX identifiers. A licence expression is allowed when every identifier in it is on this list. A licence not on it MUST NOT be used until an RFC adds it.

- `ISC`
- `MIT`
- `BSD-2-Clause`
- `BSD-3-Clause`
- `Apache-2.0`
- `Apache-2.0 WITH LLVM-exception`
- `Unicode-3.0`
- `Zlib`
- `BSL-1.0`
- `0BSD`
- `Unlicense`
- `IJG`
- `NCSA`
- `CDLA-Permissive-2.0`
- `CC0-1.0`
- `CC-BY-4.0`
- `CC-BY-SA-4.0`
- `OFL-1.1`
- `Ubuntu-font-1.0`
- `Bitstream-Vera`
- `LicenseRef-Public-Domain`

Each keeps its notice: you MUST record it in `ATTRIBUTION.md` and `NOTICE`. For `IJG`, you MUST also state that the software is based in part on the work of the Independent JPEG Group.

Language data in storytold/craft-i18n also uses `LicenseRef-SCOWL`, `LicenseRef-BSD-Lebedev` and `LicenseRef-hyph-bg`. RFC-0026 left these for their own decision, and this list defers them too: they stay allowed for that language data until an RFC decides, and for nothing else.

A repository MAY enforce a shorter list in its configuration (`deny.toml`); its `## This repository` section states that. A repository SHOULD NOT add an identifier.

An `OFL-1.1` font is allowed whoever designed it, if it comes from its open-source release and not from a copy a product installed. One with a Reserved Font Name is used unmodified; a subset or other change needs a new name.

## Copied and vendored code

Vendored or ported code MUST keep its upstream copyright and licence notice and MUST get an `ATTRIBUTION.md` row: source URL, commit and licence. Apache-2.0 code MUST keep its `NOTICE`, and files we change MUST say so. Its licence MUST be on the list above.

## MPL-2.0

`MPL-2.0` is allowed for third-party dependencies used unmodified. `MPL-2.0+` counts as `MPL-2.0`. Its copyleft applies file by file:

- Depending on an unmodified MPL-2.0 crate needs no separate process or library. Our code that uses it keeps its own licence.
- You SHOULD NOT copy MPL-2.0 code into our own files: that file would become MPL-2.0. Copying, vendoring or patching MPL-2.0 code, including through a `[patch]` entry in `Cargo.toml`, SHOULD have a maintainer's approval on the pull request first.
- A patched MPL-2.0 file's source MUST be published under MPL-2.0. Keep the patch in a public repository and link it in the pull request.
- A pull request that adds an MPL-2.0 dependency SHOULD name the crate and the apps that ship it.
- A package that ships MPL-2.0 code MUST tell users where to get that code's source and MUST keep its licence and notice files.

## Trademarks

`LicenseRef-ArtCraft-Trademark` is not on the list. It covers only the ArtCraft name and logos in `docs/brand/`, under `docs/brand/LICENSE-brand.txt`, as [clean-room.md](clean-room.md) describes. Attribution files written under the house rules call the same marks `LicenseRef-Storyteller-Trademark` (pdfcraft's `ATTRIBUTION.md` does today). Both mean the same marks; each repository renames the identifier in its rollout pull request.
