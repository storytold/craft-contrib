# RFC-0097: Inclusion of TeX Live's data files

## Summary

Allow [TeXCraft](https://github.com/skadge/texcraft) to depend on the crate
`retex-data`. The crate carries an archive of unmodified TeX Live packages,
compiled into the app, under the licences TeX Live accepts for them: LPPL, the
GUST Font License, Knuth's terms, GPL and LGPL. They are allowed as **TeX Live
data**, a class of their own like the language data of
[RFC-0011](0011-language-data-in-craft-i18n.md), and not added to the allowlist
for code, assets or fonts.

## Motivation

retex, the TeX engine TeXCraft is built on, adds a cargo feature `bundled`
in its 2.6 release. The feature is backed by the crate `retex-data`
(2026.0.0), whose archive holds 64 TeX Live 2026 packages (9.7 MB): the
LaTeX kernel, the AMS packages, graphics, hyperref, babel's English, TikZ,
fontspec and unicode-math with Latin Modern. With it, the desktop app
typesets common documents with no TeX Live installed and no network. Today
a first-time user must install TeX Live, a multi-gigabyte download, or wait
for each package to download.

The crate's code is CC0-1.0, but the archive's files keep their TeX Live
licences, and `standards/licences.md` allows none of LPPL, GPL, LGPL, the
GUST Font License or Knuth's terms. `AGENTS.md` rule 26 therefore blocks the
dependency until an RFC decides.

Out of scope: packages a user's machine fetches at run time, from TeX Live's
mirrors or from a server of TeXCraft's. They are not distributed with the
app, so they are not a dependency, asset or vendored code.

## What the archive holds

From `retex-data` 2026.0.0's `MANIFEST.toml`:

| Licence (TeX Live's catalogue) | Packages | Examples |
|---|---|---|
| LPPL 1.3c, 1.3, 1.2 | 46 | the LaTeX kernel, amsmath, graphics, hyperref, babel, fontspec, unicode-math, xcolor |
| GPL (2, 3, or either) | 5 | luaotfload (LuaLaTeX's font loader), pgf/TikZ (LPPL 1.3c or GPL-2), preview, pdftex's and mptopdf's TeX files |
| Knuth's terms | 3 | Computer Modern, plain TeX's files, BibTeX's styles |
| GUST Font License | 2 | Latin Modern, Latin Modern Math |
| LGPL-2.1 | 1 | kpathsea's `texmf.cnf` (a configuration file) |
| OFL-1.1, public domain, Unicode's licence | 4 | the AMS fonts, graphics-cfg, tex-ini-files, unicode-data |
| The terms in the files' headers (no catalogue entry) | 3 | hyphen-base, hyphen-english, latexconfig |

Every file is byte-identical to TeX Live's package archive at a locked
revision: `core.lock` holds each package's revision and checksum. The
archive carries each licence's text in `LICENSES/` and, for the GPL and LGPL
packages, their TeX Live source containers in `sources/`. It contains no
documentation.

## The licences

- **LPPL** (1.3c, 1.3, 1.2): unchanged files may be copied and distributed
  with the licence. A changed file must say so and, for many packages, take
  another name. The archive holds only unchanged files.
- **GUST Font License**: LPPL's terms for fonts (Latin Modern).
- **Knuth's terms**: free to copy and use. A changed file must not keep its
  name.
- **GPL-2.0, GPL-3.0**: these are programs in TeX's and Lua's languages,
  which the engine runs as input; nothing is linked into TeXCraft's code.
  The archive is extracted at run time into the user's cache, and the files
  are read from there as separate files. That is aggregation, which both
  GPL versions permit. The obligations are the licence text and the
  complete source, and the archive ships both.
- **LGPL-2.1**: a configuration file, read as data, shipped with its text
  and source.

None of these licences reaches TeXCraft's own code, or retex's.

## Decision

1. **TeX Live data.** Files of TeX Live's packages may be shipped as an
   archive compiled into an app, or as a file beside it, when they are:
   - unchanged, byte-identical to TeX Live's archive at a locked revision;
   - used only as input to a TeX engine.

   They may then be under these licences, as well as the allowlist's:
   `LPPL-1.3c`, `LPPL-1.3a`, `LPPL-1.2`, `LicenseRef-GUST-Font-License`,
   `LicenseRef-Knuth`, `GPL-2.0-only`, `GPL-2.0-or-later`, `GPL-3.0-only`,
   `LGPL-2.1-only`, and the free terms in a file's own header where TeX
   Live's catalogue has no entry. These identifiers are not added to the
   allowlist for code, assets or fonts.
2. **Conditions.** The archive:
   - MUST come with a lock of each package's revision and checksum;
   - MUST carry each licence's text and a manifest of each package's
     licences;
   - MUST carry the TeX Live source containers of its GPL and LGPL packages.

   Its files MUST NOT be changed: fixes go upstream, and the pin moves. The
   app records the archive in `ATTRIBUTION.md` and `NOTICE`: the crate, its
   version, TeX Live's year, and a pointer to the manifest.
3. **The dependency.** TeXCraft may depend on `retex-data` through retex's
   `bundled` feature, in its desktop app and CLI. `retex-data` 2026.0.0
   meets point 2. A new version of it is a dependency update, reviewed as
   any other: its manifest must still meet points 1 and 2.
4. **Clean room unchanged.** `standards/clean-room.md` still forbids agents
   to read GPL, LGPL or AGPL source. Depending on `retex-data` doesn't let
   an agent read its GPL packages (luaotfload, preview, the pdftex and
   mptopdf files). pgf may be read under its LPPL licence.
5. **Where the rule lives.** `standards/licences.md` gets a "TeX Live data"
   section with points 1, 2 and 4, after its paragraph on language data.

## Alternatives

- **Leave out the GPL and LGPL packages.** LuaLaTeX would lose its fonts.
  luaotfload is how LuaLaTeX loads every OpenType font, and fontspec and
  unicode-math depend on it.
- **No bundle; download at run time only.** This is what TeXCraft does
  today. A first document then needs a network and several minutes, and an
  offline machine without TeX Live can't typeset at all.
- **Add the identifiers to the allowlist.** That would allow GPL and LPPL
  code and assets in every app, which this RFC does not want.
- **A data repository of the suite's own** (like craft-fonts): `retex-data`
  already pins, locks, checks and documents the files. A copy would add a
  sync job and nothing else.

## Open questions

- **Compiled in, or beside the app.** retex also finds the bundle as a file
  beside the executable or in `/usr/share/retex/`. Shipping it as a separate
  file in the app's packages is the plainest aggregation. Compiling it in
  keeps the app one file. This RFC allows both; should it prefer one?

## References

- retex-data 2026.0.0 (crates.io): `README.md`, `LICENSING.md`, `core.txt`,
  `core.lock`, and the archive's `MANIFEST.toml`.
- TeX Live's licensing: <https://tug.org/texlive/LICENSE.TL>.
- TeXCraft: `docs/ROADMAP.md` W14.
- [RFC-0011](0011-language-data-in-craft-i18n.md) (language data, the model
  for a data class), [RFC-0027](0027-allowlist-matches-dependencies.md) (the
  allowlist and what the apps depend on).
