# RFC-0044: Add ToonCraft to the suite

This RFC builds on [RFC-0000](https://github.com/storytold/craft-contrib/blob/main/rfcs/0000-decision-process.md),
under which adding an app is an RFC with a 7-day comment period. Its sections follow the outline of
RFC-0041. File to add: `rfcs/0044-app-tooncraft.md`.

## Name and id

- Display name: **ToonCraft**.
- Machine id: `tooncraft`. Binaries: `tooncraft` and `tooncraft-cli`.
- App id: `ai.storyteller.tooncraft`.
- The name check is recorded in the repository's `docs/feature-names.md`.

## What it is

ToonCraft is a raster, hand-drawn, frame-by-frame animation app. You draw with pressure-sensitive
brushes on a tiled canvas and time the drawings on an exposure sheet. The onion skin and light table
show neighbouring drawings, and a paint bucket with gap closing fills colour under the lines. You
play the shot against its soundtrack and export to the editor or the compositor.

It is a desktop app (egui) and a CLI built from one Rust codebase. It is for animators, students
and studios. It is like TVPaint Animation, the drawing side of Toon Boom Harmony, Clip Studio
Paint's animation tools and Procreate Dreams. Those names appear only to say what ToonCraft is
like (see RFC-0029).

Repository: https://github.com/XmYx/tooncraft (MIT OR Apache-2.0). Version 0.1.0, not released yet.

## Fit

It is its own app: nothing else in the suite animates hand-drawn raster frames.

- **AniCraft** animates vector artwork with tweens and symbols. ToonCraft is its raster
  counterpart, and the two should share crates where they meet, such as timing and export.
- **PhotoCraft** paints single images.
- **Film pipeline:** boards from BoardCraft (RFC-0043) become ToonCraft shots. Finished shots go to
  FilmCraft (video and per-layer image sequences) and to EffectCraft (ProRes 4444 with alpha).

## Where it is today

These numbers are for `main` at 5f27426 and come from the repository; work since then is listed below. `cargo xtask ci` passes on GitHub. It runs fmt,
clippy, 134 tests (core, property, interop round trips, CLI end to end and 15 kittest UI flows), and
the layering, asset, licence (516 packages), progress, parity and string checks.

- **Canvas:** tiled, sparse and GPU-composited. It supports rotate and mirror, and canvases up to
  16384 px a side.
- **Brushes:** pencil with paper grain, ink, airbrush, chalk, eraser, image tips and presets. They
  respond to pressure, tilt and the eraser end on Linux X11, macOS and Windows. Speed dynamics and a
  stabilizer are included.
- **Selection and shapes:** lasso and rectangle selections you can move, scale, rotate and flip live;
  lines, rectangles and ellipses.
- **Exposure sheet:**
  - Layers with keys, holds and blanks.
  - Re-timing on 1s–4s without losing drawings.
  - Copy, cut and paste of ranges across layers, linked or as new drawings.
  - Blend modes, folders and thumbnails.
- **Camera, sound and playback:** eased camera keys, and a soundtrack with a waveform and frame
  scrubbing. Playback is clock-driven at the shot's frame rate.
- **Onion skin and light table:** before and after counts, tints and falloff; any frame can be
  pinned under the drawing.
- **Interchange:** all built in, with no external tools.
  - Layered PSD/PSB and OpenRaster, in and out.
  - XDTS exposure sheets, in and out.
  - ProRes 4444 with alpha, per-layer PNG sequences, MP4 (H.264 + AAC), GIF and APNG, out.
- **Data safety:** unlimited, memory-bounded undo; journaled atomic saves with a backup and saved
  versions; background autosave with crash recovery; a property-tested loader.
- **Measured** (`docs/performance.md`):
  - A 4K, 240-drawing shot plays at 24.1 fps with 0 dropped frames and loads in 0.12 s.
  - Ink stroke latency is 0.008 ms per pen sample on average (0.020 ms at the 99th percentile).
- **Since 5f27426:** pegs and transform layers, multiplane depth and camera shake (merged). Effects and sound, timing tools and layers, and the brush editor and export breadth are in progress as PRs on the repository.
- **Parity:** 78% of 134 feature rows at 5f27426 (`cargo xtask parity`). Weighted by effort it is about 35%,
  because effects, lip-sync and pegs were each one row but most of the remaining work.
  - `docs/user-research.md` lists what users of the reference apps complain about and ask for.
  - `docs/gaps.md` ranks what is left. Wayland-native pen input, a non-destructive effects stack,
    and lip-sync assist led it at 5f27426.

## Screenshots

![tooncraft onion skin](https://raw.githubusercontent.com/XmYx/tooncraft/main/docs/images/tooncraft-onion-skin.png)
![tooncraft timeline](https://raw.githubusercontent.com/XmYx/tooncraft/main/docs/images/tooncraft-timeline.png)

## Standards

- **House rules:** the house-rule, Rust and accessibility blocks in `AGENTS.md` are generated from
  craft-contrib.
- **Never crash:** never-crash lints on every production crate root, and no `unsafe`.
- **Accessibility:** every control is named, including the canvas and the exposure-sheet cells.
  Everything works from the keyboard, including drawing with a keyboard pen. The 15 kittest UI
  flows run with the accessibility check on. It has not been tested with a screen reader.
- **Localization:** strings go through the `craft-i18n` crate and `CRAFT_I18N_DIR`, and
  `cargo xtask` keeps the source-string catalog up to date. No ToonCraft catalog exists in
  craft-i18n yet.
- **Release setup:** `[workspace.package]` versioning, `packaging/env.sh`, and CI and release callers
  to `craft-common-actions`, set up as the suite apps have them.
- While the repository lives outside storytold, the SonarCloud and release jobs carry `if: github.repository_owner == 'storytold'`, so they stay inert until a transfer. Everything else runs as in the sibling apps.
- **Shared code first (RFC-0040):** it uses these craft-libs crates:
  - `craft-psd`, `craft-ora`, `craft-gif`, `craft-image-codecs`;
  - `craft-video`, `craft-audio`, `craft-media`;
  - `craft-keymap`, `craft-zip`, `craft-atomic`, `craft-i18n`.

## Clean room

ToonCraft was built from public documentation, manuals, tutorials, file-format specifications and
the observed behaviour of the reference apps. No code was read or copied from any animation tool,
including GPL ones such as OpenToonz, Krita, Pencil2D and Synfig. No strings, brushes, icons or
artwork came from any animation product, and distinctive feature names were replaced with our own.
Every asset has a row in `ATTRIBUTION.md`. `NOTICE` says the project is not affiliated with the
makers of the reference apps.

## Maintainers

@XmYx (author and maintainer so far).

## Prototype

The repository above.

## Open questions

- Does the app move to `storytold/tooncraft`, or stay at its current home and join the launcher
  catalog from there?
- AniCraft and ToonCraft: which timing, exposure and export code becomes a shared craft-libs crate
  first?
- The raster brush engine: should it become a craft-libs crate that BoardCraft and PhotoCraft can
  use too?
