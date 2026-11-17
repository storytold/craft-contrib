# RFC-0042: Add ScriptCraft to the suite

Builds on [RFC-0000](https://github.com/storytold/craft-contrib/blob/main/rfcs/0000-decision-process.md),
under which adding an app is an RFC with a 7-day comment period. The sections follow the outline of
RFC-0041. File to add: `rfcs/0042-app-scriptcraft.md`.

## Name and id

Display name **ScriptCraft**, machine id `scriptcraft` (binaries `scriptcraft` and `scriptcraft-cli`),
app id `ai.storyteller.scriptcraft`. The name check is recorded in the repository's
`docs/feature-names.md`.

## What it is

A screenwriting app: write a screenplay in industry-standard format, with paging that matches what a
production expects, and hand the script on to the rest of the film pipeline. Desktop app (egui) and
CLI from one Rust codebase. For screenwriters, students and productions. The familiar apps it is like:
Final Draft and its competitors, named only to say what ScriptCraft is like (see RFC-0029).

Repository: https://github.com/XmYx/scriptcraft (MIT OR Apache-2.0). Version 0.1.0, not released yet.

## Fit

It is its own app: nothing in the suite writes screenplays. It overlaps no existing app (WordCraft is
a general word processor; screenplay paging, element flow and revisions are their own product). It is
the first step of a film pipeline the suite does not have yet: ScriptCraft → BoardCraft
(RFC-0043, storyboards) → FilmCraft. ArtCraft's roadmap of narrative shorts starts with a script.

## Where it is today

Numbers for `main`, from the repository. `cargo xtask ci` passes: fmt, clippy, 114 tests, the
layering, asset, licence (510 packages) and string checks, and the progress fragments.

- Fountain open/save (lossless), pagination for US Letter and A4 with MORE/CONT'D, automatic CONT'D
  and dual dialogue; PDF export with exact pages and embedded fonts.
- FDX open and save: elements, scene numbers, dual dialogue, title page. Styled text and notes are
  not yet carried.
- Editor: Enter/Tab element flow, auto-complete, find and replace, spell-check, scene navigator,
  index cards, page preview; fast on feature-length scripts.
- Production: locked scene numbers with A/B numbering, revision marks and colours; reports and
  statistics.
- Data safety: autosave with recovery, backups, atomic saves.
- A scene list for BoardCraft (JSON with stable scene ids, documented in `docs/interop.md`).
- Parity: 82% weighted against the industry's screenwriting apps (`docs/target-app-parity.md`,
  scored by `cargo xtask parity`). What users of those apps complain about and ask for is in
  `docs/user-research.md`; the ranked remainder is `docs/gaps.md` (anchored notes, title-page
  layout, mid-element selection and FDX styled text lead it).

## Screenshots

![editor](https://raw.githubusercontent.com/XmYx/scriptcraft/main/docs/images/editor.png)
![index cards](https://raw.githubusercontent.com/XmYx/scriptcraft/main/docs/images/index-cards.png)
![revision mode](https://raw.githubusercontent.com/XmYx/scriptcraft/main/docs/images/revision-mode.png)
![pdf export](https://raw.githubusercontent.com/XmYx/scriptcraft/main/docs/images/pdf-export.png)

## Standards

- House rules, Rust and accessibility blocks in `AGENTS.md`, generated from craft-contrib.
- Never-crash lints on every production crate root; no `unsafe`.
- Accessibility: named script lines and cards, live regions, page text nodes; 16 kittest keyboard
  tests with the accessibility check on. Tree tests do not prove screen-reader support.
- Localization through the `craft-i18n` crate and `CRAFT_I18N_DIR`; every interface string is
  translatable. No ScriptCraft catalog exists in craft-i18n yet.
- `[workspace.package]` versioning, `packaging/env.sh`, nfpm and desktop files, and CI and release
  callers to `craft-common-actions`. A draft launcher entry is in `docs/launcher-entry.toml`.
- While the repository lives outside storytold, the SonarCloud and release jobs carry `if: github.repository_owner == 'storytold'`, so they stay inert until a transfer. Everything else runs as in the sibling apps.
- Shared code first (RFC-0040): it uses craft-libs `craft-pdf`, `craft-xml`, `craft-i18n`,
  `craft-proof` and `craft-fonts-build`.

## Clean room

Built from public documentation, file-format specifications (Fountain is an open format; FDX is read
from its published XML structure) and observed behaviour. No code, strings, templates, sample scripts
or artwork from any screenwriting product. Test fixtures were written for the project. Every asset
has a row in `ATTRIBUTION.md`, and `NOTICE` says the project is not affiliated with the makers of the
reference apps.

## Maintainers

@XmYx (author and maintainer so far).

## Prototype

The repository above.

## Open questions

- Does the app move to `storytold/scriptcraft`, or stay at its current home and join the launcher
  catalog from there?
- Fountain and FDX parsing could become a craft-libs crate, since BoardCraft reads the same formats
  (RFC-0040). Should that happen before or after the move?
- Should a ScriptCraft catalog be added to craft-i18n now, or once the interface settles?
