# RFC-0030: A shared vector drawing layer

A shared 2D geometry kernel already exists as wdunn001/craft-libs branch `shared-geometry-crates` (commit 799a7d8, "Promote shared geometry crates into craft-libs"), ported from VectorCraft. It has never landed on craft-libs main. No app has cut over to it. This RFC gives that work the suite-level decision it needs, per the admission gates proposed in RFC-0009 (storytold/craft-contrib#10).

## Summary

The suite gains one 2D vector drawing kernel in craft-libs: `craft-geom` holds the editable path model (anchor-based `PathData`, shape generators, hit testing, live corners, shape recognition) over kurbo; `craft-pathops` holds the path algebra (booleans over linesweeper, Pathfinder operations, Live Paint planar maps, offset, outline stroke, simplify); `craft-trace` turns rasters into `PathData`; `craft-testkit` carries the property-test strategies and geometry assertions that keep the other three honest, behind a `geom` feature. VectorCraft, FigCraft and LaserCraft are the committed consumers. Snapping and the anchor data model are named as the next extraction seams. The interactive tool layer and the canvas stay in the apps for now.

## What exists today, honestly assessed

The kernel is written, tested and unlanded. The four crates on `shared-geometry-crates` total roughly 19,000 lines with about 4,600 lines of property and regression tests. They forbid unsafe code and depend only on kurbo, linesweeper and image. They are absent from craft-libs main. They have no CI workflows, no crate READMEs and no entry in the root README.

Nobody consumes them fully. VectorCraft, the crates' own source, still builds against private path dependencies (`vectorcraft-geom`, `vectorcraft-pathops`, `vectorcraft-trace`) while pinning its other craft-libs crates at one rev (9be9a637). FigCraft carries its own `figcraft-geom` (about 760 lines) and `figcraft-pathops` (about 1,600 lines, also over linesweeper) while taking its craft-libs crates (`craft-ui`, `craft-cloud`, `craft-docpath`, `craft-window`) as floating `branch = "quasarke"` dependencies on a personal fork. LaserCraft is the only consumer of the kernel today, through a five-function wasm seam (weld, boolean, difference or intersect, offset, concentric fill) in its `geomops` adapter, pinned to the feature branch, while its `craft-atomic` and `craft-settings` dependencies come from a different branch.

The duplication above the kernel is wide. Snapping and smart guides exist three times: VectorCraft's `tools/src/guides` (about 1,800 lines covering point, grid, pixel, glyph, tangent and perpendicular targets), FigCraft's `snap.rs` plus `point_snap.rs` (about 540 lines, a strict subset) and LaserCraft's JavaScript `alignSnap`. Path booleans over linesweeper exist twice with zero shared code. The same concepts carry divergent types: four private `Handle` enums, two `NodeKind` enums, three different answers to what a tool is. LaserCraft's entire editor is about 4,500 lines of JavaScript in one file. That file reimplements, on flat polylines, what `craft-geom` already provides as curves: shape generators, grid snap, hit testing, fillet and chamfer corners, angle-constrained drag, arc-length text on a path. Its source comments name the pattern plainly ("Mirrors SolidCraft's shared modifier resolver", "ported from SolidCraft's sketch-modify set"). LaserCraft also keeps local `svg` and `dxf` reader crates although craft-libs main now ships `craft-svg` and `craft-dxf`.

The cost of extract-without-adoption is already visible in the tree. `crates/affinity` exists in both craft-libs and VectorCraft with an identical fifteen-file layout and every file diverged (7,874 against 7,346 lines). The kernel will drift the same way if it lands without cutover commitments.

## The kernel branch has fallen behind its source

`shared-geometry-crates` is a snapshot of VectorCraft's geometry crates taken on 2026-10-22 (799a7d8604). VectorCraft has kept developing them since. Measured on 2026-11-15 against VectorCraft main at cad4bb0c6e, 270 commits have touched the four kernel source trees since the snapshot: 42 in `crates/geom/src`, 104 in `crates/pathops/src`, 103 in `crates/trace/src`, 23 in `crates/testkit/src`.

The divergence is uneven across the four. In `geom`, 9 of the snapshot's 12 files differ from their current source. VectorCraft has also added `src/anchors.rs`. In `pathops`, the snapshot has 14 files against VectorCraft's 45. Thirty-six VectorCraft files have no counterpart in the snapshot, among them the `addons` path tools, cleanup, keep-shape, dashes, extend, crossings and shared-edge code. In `trace`, the snapshot has 38 files against VectorCraft's 50, missing deblock, layers, opaque, params, pipeline, raster and the quantiser's denoise, palette, rare-colour and cut-edge passes. In `testkit`, the snapshot carries `geom.rs` and `strategies.rs` alone. VectorCraft's `testkit` has since grown helpers for PDF, TIFF, `.ase`, Illustrator, fonts, fixtures, rasters and invariants. Those belong in RFC-0035's crate.

One of those commits is a hostile-input limit. storytold/vectorcraft@b12ee928f bounds how many anchors serde reads for a set of paths (`with_anchor_budget`, `anchors_within`, `MAX_READ_ANCHORS`, 32M anchors native and 4M on wasm32). A small compressed file of repeated anchors expands to gigabytes once read, past what the web build can address. House rule 1 requires that bound. The snapshot does not carry it. A cutover measured by deletion of the private copy would delete the limit, unless step 1 re-syncs first.

## Why a shared layer

Every drawing app in the suite needs the same math: an editable bezier path model, booleans, offsetting, hit testing, snapping. Each app growing its own copy reproduces the same algorithms with drift, as the affinity fork already demonstrates. The kernel is pure algorithms and data over kurbo, with no UI, no I/O and no app types. RFC-0009 admits this shape by name: reusable data models, algorithms and contracts. The kernel also satisfies RFC-0009's default threshold with room to spare: three apps consume or reimplement it today.

## Architectural boundaries

The kernel sits below everything. `craft-geom` and `craft-pathops` depend on kurbo and linesweeper only, never on egui, `craft-ui` or any app crate. The existing craft-ui layer contract stays intact: only app UI crates depend on `craft-ui`, never document or engine crates. Apps keep their document models, their appearance systems (fill, stroke, text, layers), their tool dispatch and their canvases. The kernel's `PathData` is the interchange type at those seams. Interchange crates already on main (`craft-svg`, `craft-dxf`, `craft-gcode`, `craft-plotter`) are unaffected. The names `craft-sketch` and `craft-svg-paths` on main are format crates with unrelated purposes (a `.sketch` container reader and a compatibility re-export). The kernel's names collide with nothing.

## What this RFC decides

**1. The kernel lands on main, re-synced from VectorCraft.** The four kernel crates move from `shared-geometry-crates` onto craft-libs main, rebased to current workspace lints, each with a README, a CI workflow and a root README entry. The landing PR re-syncs the three ported crates (`craft-geom`, `craft-pathops`, `craft-trace`) from one named VectorCraft rev, recorded in the PR. VectorCraft's tests for those crates come across in the same change. `craft-testkit` takes VectorCraft's geometry test helpers only, under the `geom` feature of decision 5. Per RFC-0009, each crate has a named responsible maintainer: wdunn001. A second named reviewer per crate is sought from VectorCraft and FigCraft during the comment period. Step 1 does not wait on one.

**2. Consumers cut over by pinned rev, deleting their copies.** VectorCraft retires its three private path crates in favour of the shared ones at its existing single-rev pin discipline. FigCraft retires `figcraft-geom` and `figcraft-pathops`, migrating its node model's bezier representation onto `craft_geom::PathData`. FigCraft also repoints its floating branch dependencies to pinned revs on the storytold repository. LaserCraft consolidates its split pins (feature branch for geometry, `quasarke` for the rest) onto one pinned main rev; widening its wasm seam beyond the current five functions is app work it can then do at will. An adopting PR is complete only when the app's private copy is deleted in the same change.

**3. Snapping is the next seam.** A shared snapping and smart-guide crate is extracted from VectorCraft's `tools/src/guides`, the superset implementation, as geometry in and guide results out, with no egui types in its API. FigCraft and LaserCraft adoption is the acceptance test. SolidCraft's `drag_snap` is a possible fourth consumer later.

**4. The anchor model converges.** FigCraft's node, handle-mirroring and region types converge on `craft-geom`'s `Anchor` and `PathData`, with round-trip fidelity proven by property tests in both directions. The property tests cover the serialised form as well as the in-memory conversion. A document written before the convergence then reads back unchanged after it. This removes the last private bezier model among the committed consumers.

**5. The geometry test helpers sit behind a `craft-testkit` feature.** The geometry strategies and assertions (`strategies.rs` for paths and shapes, `geom.rs` for Hausdorff closeness) live behind a `geom` feature of `craft-testkit`. That feature alone pulls in the geometry dependencies: `craft-geom`, `kurbo`, `proptest`. The crate's default build therefore depends on no craft-libs crate. The kernel crates take the feature from their integration tests under `tests/` only, never from `#[cfg(test)]` unit tests. A unit test taking it would build a second `craft-geom` whose types do not match the crate under test. RFC-0035 (storytold/craft-contrib#55) proposes the rest of `craft-testkit` and states the same contract for this feature. Whichever of the two RFCs lands first creates the crate. The other adds its modules to it. craft-libs main has no `crates/testkit` and none of the kernel crates today. Either order works. The code behind the feature is the kernel's own test support. This RFC is therefore the source for what the `geom` feature holds.

## Acceptance requirements

**Kernel CI proves the algebra, every merge.** The existing proptest suites (path round-trips, boolean postconditions, Hausdorff closeness in `craft-testkit`) run in craft-libs CI for every kernel crate, with the `geom` feature enabled for the suites that need it. A kernel crate with red or absent CI blocks adoption PRs.

**Cutover is measured by deletion.** Decision 2 counts as done per app when the app builds with no local geom, pathops or trace crate and no floating branch dependency on craft-libs. Review of each adoption PR checks for behavioural regressions against the app's existing tests, with any intentional divergence from the private copy written down in the PR. Each adoption PR also records the VectorCraft rev the shared crates were re-synced from. It shows that the shared code and the deleted private code agree. A remaining difference is listed with its reason.

## Compatibility through the cutover

`craft_geom::PathData`'s serde form is a published format. VectorCraft's `.vectorcraft` documents store anchors in it. VectorCraft's command API parses it: `crates/engine/src/cmd/stroke.rs:261` reads a custom arrowhead's outline with `serde_json::from_value::<PathData>`. The moment VectorCraft deletes its private `geom` crate, the kernel's serde attributes become that file format and those command parameters.

The two forms agree today. `Anchor` serialises through `AnchorRepr` in both copies. That form writes `p` as `[x, y]`. It omits `in` and `out` when they equal `p`. It omits `kind` at its default. The block is identical in the snapshot and in VectorCraft's head. The re-sync of step 1 does not move it. The differences in `path.rs` are the anchor read budget described above and `SubPath::anchors` holding VectorCraft's chunked `Anchors` type. Neither one changes the wire form. The cutover holds the agreement in place.

- The serde form of the path types stays the v2-and-later `.vectorcraft` anchor form (`PathData`, `SubPath`, `Anchor`, `AnchorKind`). A later change to it is a file format version with a migration, through the mechanism `vectorcraft-format` already documents. Its v1 wrote `{p: {x, y}, in, out, kind}` and its v2 writes `{p: [x, y], in?, out?, kind?}`. Its `save_with` writes any version from `MIN_VERSION` to `VERSION` for older apps.
- VectorCraft's public command parameter names and accepted values stay as they are. A rename is a breaking change to the command API, carried by its own PR with its own note. It is never implicit in a crate move. Upstream refuses unknown parameters. A fork that carries patches cannot rebase over a silent rename.
- FigCraft's saved documents stay readable across its adoption PR, by migration where its node model's persisted form changes.

The VectorCraft adoption PR proves the first two. It loads pre-cutover fixture documents of every file version the app supports. It compares each loaded document against the pre-cutover result. It replays pre-cutover command calls that take a `PathData` parameter unchanged, `stroke.arrowheads` custom heads among them.

## Reference implementations, non-normative

VectorCraft's `crates/tools` and `crates/pathops`, FigCraft's `crates/engine` vector-edit modules and LaserCraft's web editor plus `geomops` adapter informed this design and serve as reference material. None of them is a dependency of the shared layer.

## Landing order

1. Re-sync `shared-geometry-crates` from the named VectorCraft rev, then land it onto craft-libs main with CI, READMEs and the root README entry. Named maintainer: wdunn001.
2. VectorCraft cutover PR (decision 2). Named maintainer: wdunn001.
3. FigCraft cutover PR, including the anchor-model convergence of decision 4. Named maintainer: wdunn001.
4. LaserCraft pin consolidation PR. Named maintainer: wdunn001.
5. Snapping crate extraction and its two adoption PRs (decision 3). Named maintainer: wdunn001.

## What this RFC does not decide

A shared interactive tool layer, a shared canvas or viewport crate, a shared vector document model with appearance and a shared renderer are all named future work. The three apps currently disagree on the shape of a tool. Standardising it now would pick a winner without evidence. LaserCraft's migration off its single-file JavaScript editor is app-internal. SolidCraft adoption is out of scope: it lives in another organisation, its sketcher is a constraint solver on its own NURBS substrate with no kurbo dependency today. It is welcome later as a consumer of the geometry and snapping primitives only.

## Open questions

How suite apps pin craft-libs revs is unresolved. LaserCraft's release CI already checks a `pins.toml` that the geometry branch does not carry. The pin mechanism therefore needs one documented home before step 4. RFC-0009 is still open; this RFC conforms to its draft gates and does not depend on its merge order. Whether `craft-trace` lands in step 1 or follows separately is left to the landing PR's review. It is the largest crate with the least consumer overlap today. Its re-sync is also the biggest of the four. How much of VectorCraft's current `testkit` comes across with the `geom` feature is for the landing PR to settle with RFC-0035's author, since the rest of that crate is RFC-0035's scope.

Related to wdunn001/craft-libs@799a7d8 (the kernel branch). Context: storytold/craft-contrib#10 (RFC-0009), storytold/craft-contrib#38 (RFC-0022, the shared-layer precedent this follows), storytold/craft-contrib#55 (RFC-0035, the other RFC that names `craft-testkit`).
