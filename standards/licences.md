# Licences

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as in RFC 2119 and RFC 8174.

These rules apply to every dependency, asset, font and piece of vendored code in the Craft repositories. `AGENTS.md` carries the short form and links here. Adding an identifier needs an RFC, and only when a dependency or asset uses it.

## The allowlist

Dependencies and assets use SPDX identifiers. A licence expression is allowed when every identifier in it is on this list. A licence not on it MUST NOT be used until an RFC adds it.

- `ISC`
- `MIT`
- `MIT-0`
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

Language data in storytold/craft-i18n under `LicenseRef-SCOWL`, `LicenseRef-BSD-Lebedev` and `LicenseRef-hyph-bg` is outside this list until its own RFC decides; it stays as it is until then.

A repository MAY enforce a shorter list in its configuration (`deny.toml`); its `## This repository` section states that. A repository SHOULD NOT add an identifier.

## MPL-2.0

`MPL-2.0` is allowed for third-party dependencies used unmodified. `MPL-2.0+` counts as `MPL-2.0`. Its copyleft applies file by file:

- Depending on an unmodified MPL-2.0 crate needs no separate process or library. Our code that uses it keeps its own licence.
- You SHOULD NOT copy MPL-2.0 code into our own files: that file would become MPL-2.0. Copying, vendoring or patching MPL-2.0 code, including through a `[patch]` entry in `Cargo.toml`, SHOULD have a maintainer's approval on the pull request first.
- A patched MPL-2.0 file's source MUST be published under MPL-2.0. Keep the patch in a public repository and link it in the pull request.
- A pull request that adds an MPL-2.0 dependency SHOULD name the crate and the apps that ship it.
- A package that ships MPL-2.0 code MUST tell users where to get that code's source and MUST keep its licence and notice files.

## Vendored and ported code

Code copied, vendored or ported from another project MUST keep its upstream copyright and licence notice and MUST get an `ATTRIBUTION.md` row with its source URL, commit and licence. Apache-2.0 code MUST keep its `NOTICE`, and changed files MUST be marked as changed.

## Fonts

1. Font files SHOULD live in [storytold/craft-fonts](https://github.com/storytold/craft-fonts). New fonts SHOULD be added there and SHOULD NOT be added to another repository. A minimal font made for a parser test MAY stay in its repository, with its provenance recorded.
2. craft-fonts is the optional build input `CRAFT_FONTS_DIR` (an absolute path) and SHOULD NOT be a `Cargo.toml` dependency. Code and tests SHOULD build and run with it unset; tests that need its glyphs SHOULD skip. Releases SHOULD build with it and ship its licences.
3. An `OFL-1.1` font is allowed whoever designed it, if it comes from its open-source release and not from a copy a product installed.
4. An `OFL-1.1` font with a Reserved Font Name MUST be used unmodified under that name. A subset or modified copy MUST get a new name.
5. A font installed on the user's machine MAY be read at runtime. It SHOULD NOT be embedded, shipped or committed in the project's own files, and SHOULD NOT appear in published screenshots. A user's own documents and exports MAY embed the fonts the user chose.

## Trademarks

`LicenseRef-ArtCraft-Trademark` is not on the list. It covers only the ArtCraft name and logos in `docs/brand/`, under `docs/brand/LICENSE-brand.txt`, as [clean-room.md](clean-room.md) describes. pdfcraft's records use the earlier identifier `LicenseRef-Storyteller-Trademark` for the same marks until pdfcraft moves to the shared `AGENTS.md`.
