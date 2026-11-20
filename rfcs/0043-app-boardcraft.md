# RFC-0043: Add BoardCraft to the suite

Builds on [RFC-0000](https://github.com/storytold/craft-contrib/blob/main/rfcs/0000-decision-process.md),
under which adding an app is an RFC with a 7-day comment period. The sections follow the outline of
RFC-0041. File to add: `rfcs/0043-app-boardcraft.md`.

## Name and id

Display name **BoardCraft**, machine id `boardcraft` (binaries `boardcraft` and `boardcraft-cli`),
app id `ai.storyteller.boardcraft`. The name check is recorded in the repository's
`docs/feature-names.md`.

## What it is

A storyboard and animatic app: break a script into scenes and panels, draw each panel, add dialogue,
timing and camera moves, play the animatic with its soundtrack, print board sheets, and hand the cut
to the editor. Desktop app (egui), CLI and MCP server from one Rust codebase. For directors, story
artists and students. The familiar app it is like: Storyboard Pro and other storyboard tools, named
only to say what BoardCraft is like (see RFC-0029).

Repository: https://github.com/XmYx/boardcraft (MIT OR Apache-2.0). Version 0.1.0, not released yet.

## Fit

It is its own app: nothing in the suite does storyboards and animatics. AniCraft animates vector
artwork and PhotoCraft edits images; BoardCraft is about panels, shots and timing. It is the middle
of the film pipeline: ScriptCraft (RFC-0042) → BoardCraft → FilmCraft. Panels are a natural place for
the suite's AI layer (`craft-ai-integrations`); a hook is in place, without an integration.

## Where it is today

Numbers for `main`, from the repository. `cargo xtask ci` passes: fmt, clippy, 145 tests, the
layering, asset and licence (567 packages) checks, and the progress fragments.

- Scenes and panels with layered pressure-aware drawing, lasso select and transform, fill, canvas
  rotation, stabilizer and pressure curve; dialogue, action and notes; durations.
- Camera moves with rotation and a multi-key editor; animatic playback with an audio track and
  waveform that stays in sync when panels are retimed.
- In: Fountain, FDX and ScriptCraft's scene-list JSON, with re-import. Out: OTIO, EDL, FCP7 XML and
  FCPXML timelines (camera moves become FilmCraft motion keyframes) and conform back; MP4 with
  burn-ins; PNG panels; PDF board sheets with selectable text.
- Remappable shortcuts through `craft-keymap`; autosave with recovery and snapshot undo.
- Parity: about 41% of the reference app, estimated (`docs/target-app-parity.md`); about 19% before
  the polish pass. User complaints and requests for storyboard tools are in `docs/user-research.md`;
  the ranked remainder is `docs/gaps.md`.

## Screenshots

![boardcraft board](https://raw.githubusercontent.com/XmYx/boardcraft/main/docs/images/boardcraft-board.png)
![boardcraft panel](https://raw.githubusercontent.com/XmYx/boardcraft/main/docs/images/boardcraft-panel.png)

## Standards

- House rules, Rust and accessibility blocks in `AGENTS.md`, generated from craft-contrib.
- Never-crash lints on every production crate root; no `unsafe`; the core crate has no UI dependency
  (checked by xtask).
- Accessibility: kittest keyboard tests with the accessibility check on; the panel grid and canvas
  publish nodes; every drag has a keyboard or numeric alternative. Not yet tested with a screen reader
  or on real tablets.
- Localization through the `craft-i18n` crate and `CRAFT_I18N_DIR`. No BoardCraft catalog exists in
  craft-i18n yet.
- `[workspace.package]` versioning, `packaging/env.sh`, and CI and release callers to
  `craft-common-actions`. A draft launcher entry is in `docs/launcher-entry.toml`.
- While the repository lives outside storytold, the SonarCloud and release jobs carry `if: github.repository_owner == 'storytold'`, so they stay inert until a transfer. Everything else runs as in the sibling apps.
- Shared code first (RFC-0040): it uses craft-libs `craft-timeline`, `craft-audio`, `craft-video`,
  `craft-media`, `craft-pdf`, `craft-image-codecs`, `craft-keymap`, `craft-settings`, `craft-atomic`,
  `craft-xml` and `craft-i18n`. Candidates to move into craft-libs: the raster brush,
  autosave/recovery, snapshot undo and the screenplay parsers.

## Clean room

Built from public documentation, interchange specifications (OTIO, EDL, FCP XML) and observed
behaviour. No code, strings, templates or artwork from any storyboard product. Every asset has a row
in `ATTRIBUTION.md`, and `NOTICE` says the project is not affiliated with the makers of the reference
apps.

## Maintainers

@XmYx (author and maintainer so far).

## Prototype

The repository above.

## Open questions

- Does the app move to `storytold/boardcraft`, or stay at its current home and join the launcher
  catalog from there?
- Which of its brush, autosave, undo and screenplay crates should move to craft-libs first, so that
  ToonCraft (RFC-0044) and ScriptCraft share them?
- Drawing: should BoardCraft keep a light raster engine, or adopt ToonCraft's or PhotoCraft's once
  they are in craft-libs?
