# RFC-0045: Add ModelCraft to the suite

Builds on [RFC-0000](https://github.com/storytold/craft-contrib/blob/main/rfcs/0000-decision-process.md),
under which adding an app is an RFC with a 7-day comment period. The sections follow the outline of
RFC-0041. File to add: `rfcs/0045-app-modelcraft.md`.

## Name and id

Display name **ModelCraft**, machine id `modelcraft` (binaries `modelcraft` and `modelcraft-cli`),
app id `ai.storyteller.modelcraft`. The name check is recorded in the repository's
`docs/feature-names.md`.

## What it is

A 3D content-creation app. You build shapes from parametric primitives with an editable build
history, add smooth cages and deformers, then sculpt million-polygon meshes with symmetric brushes
and masks. Results go to TextureCraft, MaterialCraft or ArtCraft as glTF.

The design aims are:
- an object model that is easy to learn;
- a node-based build history that stays editable;
- a sculpting mode for high-resolution meshes.

It is a desktop app (egui, wgpu viewport) and a CLI, from one Rust codebase. It is for artists who
block out, model and sculpt assets. The familiar apps it is like are Cinema 4D, Maya and ZBrush,
named only to say what ModelCraft is like (see RFC-0029).

Repository: https://github.com/XmYx/modelcraft (MIT OR Apache-2.0). Version 0.1.0, not released yet.

## Fit

It is its own app: the suite has node materials (MaterialCraft) and 3D texture painting
(TextureCraft), but nothing that makes the meshes they work on. ArtCraft stages scenes in 3D and
turns images into meshes; ModelCraft opens and saves the same glTF. It overlaps the 3D pieces in
VectorCraft and EffectCraft only through shared crates. It follows RFC-0022 (GPU layer) and is meant
to adopt RFC-0032 (shared PBR and 3D rendering crates) once accepted.

## Where it is today

Numbers for `main` (c44894c), from the repository. `cargo xtask ci` passes on GitHub: fmt, clippy, 87 unit, property and
UI tests, and the layering, asset, licence, progress and parity checks.

**Viewport**
- Orbit, pan and zoom with three navigation presets.
- Perspective and orthographic views, with single and four-view layouts.
- Move, rotate and scale gizmos, each with a numeric alternative.

**Scene**
- An object tree with groups, plus an attributes panel.
- Seven parametric primitives that stay editable after creation.
- An editable build history: change an early step and later steps re-run. Steps can be toggled,
  removed or collapsed into a mesh.
- A smooth-cage subdivision generator (0–6 levels).
- Deformers as children: bend, twist, taper, ripple and spherify.

**Modeling**
- Vertex, edge and face modes, with loop, ring, grow and shrink selection.
- Extrude, inset, loop cut, bridge, fill hole, merge/weld, delete, triangulate and mirror.
- Each operation is recorded as a history step.

**Sculpting**
- Draw, clay, smooth, inflate, grab, pinch, flatten and crease brushes, plus a mask brush.
- X/Y/Z symmetry and tablet pressure.
- Measured: 2.0 M triangles at about 280 fps while sculpting (RTX 4090), and a 0.2–0.4 ms engine
  dab on 1 M quads (`docs/performance.md`).

**Files**
- A native project format with atomic saves and autosave recovery.
- glTF/GLB, OBJ, STL and PLY through `craft-mesh`.

**Parity**
- Measured against the full products, not a subset: `docs/target-app-parity.md` maps 330 in-scope rows in 25 weighted areas against Cinema 4D, Maya, ZBrush and V-Ray/Arnold, and `cargo xtask parity` scores it in CI. Today: **14.2% weighted, 24.5% of rows**. Strongest areas: UI (62%), stability (50%), dependency graph and history (33%), polygon modeling (30%). Rendering, animation, rigging, simulation, cloning and UVs are at or near 0%.
- `docs/gaps.md` is a ranked 70-item plan toward 90%, estimated at about 2,760 hours of work. Work on the render engine, sculpting, animation and rigging, UVs, procedural modeling and UI is in progress as PRs on the repository; the numbers above are re-scored by CI on every merge.
- What users of the reference apps complain about and ask for is in `docs/user-research.md`.

## Screenshots

![modelcraft model](https://raw.githubusercontent.com/XmYx/modelcraft/main/docs/images/modelcraft-model.png)
![modelcraft sculpt](https://raw.githubusercontent.com/XmYx/modelcraft/main/docs/images/modelcraft-sculpt.png)
![modelcraft quad](https://raw.githubusercontent.com/XmYx/modelcraft/main/docs/images/modelcraft-quad.png)

## Standards

- House rules, Rust and accessibility blocks in `AGENTS.md`, generated from craft-contrib.
- Never-crash lints on every production crate root, and no `unsafe`. Mesh operations check topology
  after each step, and property tests cover them. Seven core crates have no UI dependency (checked by
  xtask).
- Accessibility:
  - every command has a keyboard path;
  - the object tree publishes tree items;
  - transforms have numeric entry;
  - kittest keyboard tests run with the accessibility check on.

  It has not been tested with a screen reader.
- Localization goes through the `craft-i18n` crate and `CRAFT_I18N_DIR`. No ModelCraft catalog exists
  in craft-i18n yet.
- Release setup:
  - `[workspace.package]` versioning;
  - `packaging/env.sh`;
  - CI and release callers to `craft-common-actions`.
- While the repository lives outside storytold, the SonarCloud and release jobs carry `if: github.repository_owner == 'storytold'`, so they stay inert until a transfer. Everything else runs as in the sibling apps.
- Shared code first (RFC-0040): it uses craft-libs `craft-mesh`, `craft-zip`, `craft-image-codecs`,
  `craft-keymap`, `craft-settings`, `craft-atomic` and `craft-i18n`.

## Clean room

Built from public documentation, manuals, tutorials and observed behaviour of the reference apps.
No code was read or copied from any 3D application, including GPL ones such as Blender. No strings,
icons or artwork come from any 3D product, and distinctive feature names were replaced with our own.
Every asset has a row in `ATTRIBUTION.md`. `NOTICE` says the project is not affiliated with the makers
of the reference apps.

## Maintainers

@XmYx (author and maintainer so far).

## Prototype

The repository above.

## Open questions

- Does the app move to `storytold/modelcraft`, or stay at its current home and join the launcher
  catalog from there?
- RFC-0032: should ModelCraft's viewport renderer wait for the shared PBR and 3D crates, or offer its
  half-edge mesh and viewport as a starting point for them?
- Hand-off to TextureCraft and MaterialCraft: glTF only, or a shared live-link once both sides have
  UVs?
