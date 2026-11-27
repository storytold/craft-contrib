# RFC-0050: Shared crates for PBR materials and 3D rendering

This RFC revises the 2026-10-27 design by addibble
(<https://gist.github.com/addibble/0895ec701abaa70ac8f4bf6154750317>), which proposed moving
EffectCraft's physically based renderer into new craft-libs crates. Since that design, TextureCraft and
MaterialCraft have added their own PBR code, craft-libs has added mesh, MaterialX, OpenEXR and Radiance
HDR crates, and RFC-0022 has decided the GPU layer. This revision plans the move with those changes.

A second revision, on 2026-11-26, widens the scope from the four apps the first design named to every
Craft app. It adds "Scope", the status of the crates in craft-libs on that date, and Appendix A, which
lists the 3D code each app has.

Related: storytold/craft-contrib#59, storytold/craft-libs#778, storytold/craft-libs#115,
storytold/vectorcraft#2560, storytold/effectcraft#2970, storytold/effectcraft#2199,
storytold/deckcraft#743, storytold/deckcraft#1380, storytold/photocraft#8805, storytold/cadcraft#1796.

## Summary

Three new craft-libs crates, `craft-mesh-ops`, `craft-pbr` and `craft-raster3d`, and the existing
`craft-texture` hold the 3D code that the apps carry separately. This covers every Craft app, not only
the four that carried the code when this RFC was opened. Each crate is filled from the app whose code
already has tests for that part. The crates add no importer or decoder of their own. They read the output of
`craft-mesh`, `craft-materialx`, `craft-hdr`, `craft-exr` and `craft-raster-codecs`, and the decoding
features of `craft-texture` call those format crates. VectorCraft is a planned consumer.

## Scope

This RFC covers every Craft app. Code that draws, shades, lights or prepares 3D geometry and textures,
and that more than one app has or will need, goes to one of the four crates. An app keeps what belongs to
its product: its user interface, tools, importers, document model, effect parameters and the effects
built on the shared code. Appendix A lists each app's 3D code, the crate it belongs in and which copy is
the reference. It was read on 2026-11-26 and covers every repository named there. A repository with no
row has no 3D code in the patterns searched.

1. **Four crates.** The split is `craft-mesh-ops`, `craft-texture`, `craft-pbr` and `craft-raster3d`.
   Code from another app goes into the crate it fits. A fifth crate needs a pull request that names the
   code that fits none of the four and the apps that will use it. There is no catch-all geometry crate
   (see "Defaults this RFC takes", default 1).
2. **Reference copy.** Where several apps carry the same code, the reference is the copy with the most
   tests, as in "The crates". Differences between copies are separate output-change pull requests with
   the measured difference.
3. **Owners.** Each app's maintainers own that app's switch pull request. This RFC does not change another
   app's code.
4. **Invitations.** The comment period's start names the apps in Appendix A. RFC-0019 asks for a review
   request to each affected app's maintainers.

## What existed on 2026-11-13

Read on 2026-11-13 at craft-libs `75c70fa6`, effectcraft `05fc7f28`, texturecraft `386630ee`,
materialcraft `31330629`, pdfcraft `0f22627b` and vectorcraft `08b6cfd3`.

- **EffectCraft.** Cook-Torrance shading with GGX, the Smith-Schlick geometry term and Schlick Fresnel
  (`crates/render/src/three_d/adv/shade.rs`), a depth-buffered software rasterizer with shadow maps
  (`adv/raster.rs`), a default environment generated in code (`adv/scene.rs`), and a WGSL copy of the
  shading (`crates/gpu/src/shaders/advanced3d.wgsl`) that 13 tests compare with the CPU code.
  Extrusion, triangulation, primitives, normals and tangents are in `crates/model`. Its tangents are
  not MikkTSpace tangents. It reads mesh files through `craft-mesh`.
- **TextureCraft.** `crates/geom` holds a mesh type with MikkTSpace tangents (through the `mikktspace`
  crate), a BVH, UV unwrapping, a UV-space G-buffer and primitives. `crates/render/src/ibl.rs` holds
  CPU image-based lighting. The viewport and path tracer use GGX with height-correlated Smith
  visibility.
- **MaterialCraft.** `crates/geom` is a copy of TextureCraft's: `gbuffer.rs` and `unwrap.rs` are
  identical, and `bvh.rs`, `mesh.rs` and `primitives.rs` differ. `crates/render/src/ibl.rs` builds a
  GGX-prefiltered mip chain and SH9 irradiance. `crates/image` holds f32 image planes with sampling.
- **PdfCraft.** `crates/prc-render` is a CPU depth-buffered renderer with camera fitting
  (`Camera::fit`) and work ceilings.
- **VectorCraft.** `crates/three-d` draws 3D effects as shaded vector faces.
- **craft-libs.** `craft-mesh` reads mesh files into one `Scene`, and its README leaves computing
  normals, tangents and UVs to the apps. `craft-materialx`, `craft-exr`, `craft-hdr` and
  `craft-raster-codecs` exist. No crate holds textures with sampling, a BRDF, image-based lighting or a
  software 3D rasterizer.

Open pull requests add further renderers: storytold/deckcraft#743, storytold/deckcraft#1380 and
storytold/photocraft#8805. storytold/cadcraft#1796 asks for a CPU renderer for RENDER.

## What exists on 2026-11-26

Read at craft-libs `f3d15cae`, effectcraft `8087064f`, texturecraft `d422c821`, materialcraft `e7ec9cb6`,
pdfcraft `09175f49` and vectorcraft `c8c4faa3`. Appendix A names the other repositories' commits.

- **craft-texture.** storytold/craft-libs#1216 added it on 2026-11-14 with MaterialCraft's image planes,
  sampling and sRGB helpers. `main` now also holds mip chains (`mip.rs`), height to normal (`normal.rs`),
  ORM packing (`packing.rs`) and, behind a `dds` feature, DDS decoding. storytold/craft-libs#1403
  (height to normal and ORM packing) merged on 2026-11-17. Still open: storytold/craft-libs#1400 (tests
  of the sRGB pair), storytold/craft-libs#1429 (decoding PNG, TIFF, Radiance HDR and OpenEXR) and
  storytold/craft-libs#2833 (texture sets recognised from file names).
- **craft-pbr.** It is on `main`. storytold/craft-libs#2850 (GGX prefilter and SH9 irradiance),
  storytold/craft-libs#2873 (metallic-roughness BRDF, environment BRDF and the default environment) and
  storytold/craft-libs#2880 (the shading functions as WGSL text) merged on 2026-11-26. Still open:
  storytold/craft-libs#2908 (conversions from `craft-mesh` and MaterialX materials).
- **craft-mesh-ops.** It is not on `main`. storytold/craft-libs#1233 adds it, and 15 more open pull
  requests extend it (storytold/craft-libs#2842, #2898 and 13 more from #3657 to #3675).
- **craft-raster3d.** It is not on `main`. Five open pull requests add it: storytold/craft-libs#2923,
  #2950, #2958, #2985 and #2988.
- **MaterialCraft** depends on `craft-texture` (`Cargo.toml`, craft-libs rev `d7010805`) and no longer has
  `crates/image`. No app's manifest names `craft-pbr`.
- **GPU layer.** RFC-0022 is on `main` (storytold/craft-contrib#38, merged 2026-11-10) and
  storytold/craft-libs#663 merged on 2026-11-19, so `craft-gpu`, `craft-gpu-wgpu`, `craft-gpu-null` and
  `craft-gpu-providers` are in craft-libs.
- **Other apps.** DeckCraft, PhotoCraft and CADCraft have no 3D renderer on `main`. The open pull
  requests storytold/deckcraft#743, storytold/deckcraft#1380 and storytold/photocraft#8805 add 3D
  features, and storytold/cadcraft#1796 asks for a renderer. Appendix A lists the rest.

The crates' pull requests were opened on or after 2026-11-13, and the three `craft-pbr` ones merged while
this RFC was open. They follow the contents listed in "The crates".

## The crates

| Crate | Contents | Comes from | First consumers |
|---|---|---|---|
| `craft-mesh-ops` | Smooth and flat normals, MikkTSpace tangents, welding, UV unwrapping, a UV-space G-buffer, primitives with UVs, triangulation with holes, extrusion of 2D outlines with bevels. A BVH behind a `bvh` feature. Conversion from `craft_mesh::Mesh` behind a feature. | TextureCraft `crates/geom` with MaterialCraft's changes; EffectCraft `crates/model` | TextureCraft and MaterialCraft, then EffectCraft |
| `craft-texture` | A linear float texture with a color-space tag and size caps. The scalar sRGB encode and decode pair. Sampling (nearest and bilinear; repeat, clamp and mirror), mip chains, height to normal, channel packing (ORM and the glTF metallic-roughness layout) and texture-set recognition by file name. Decoding behind features that call `craft-hdr`, `craft-exr` and `craft-raster-codecs`. | MaterialCraft `crates/image`; EffectCraft's texture sampling; TextureCraft's height to normal and ORM packing | MaterialCraft and EffectCraft, then VectorCraft |
| `craft-pbr` | The material, light, environment and shadow-map data the shader reads. The BRDF and the environment BRDF. Image-based lighting from an equirectangular environment: a GGX-prefiltered mip chain and SH9 irradiance. PCF shadow lookup. Studio environments generated in code. The WGSL shading functions as text. Conversions from `craft_mesh::Material` and `craft_materialx::Material` behind features. | EffectCraft `adv/shade.rs`, the data types of `adv/scene.rs` and the shading half of `advanced3d.wgsl`; the `ibl.rs` of TextureCraft and MaterialCraft | TextureCraft and MaterialCraft for the environment; EffectCraft and MaterialCraft for the BRDF; then VectorCraft |
| `craft-raster3d` | The software rasterizer: clipping, depth buffer, opaque then sorted transparent passes, supersampling, shadow maps sized under a budget, and camera fitting. The input is a plain scene. The output is premultiplied linear RGBA with coverage, depth, normals and a part id per pixel. | EffectCraft `adv/raster.rs`; PdfCraft `Camera::fit` and its work ceilings | EffectCraft, then VectorCraft, DeckCraft, PdfCraft and CADCraft |

`craft-pbr` depends on `craft-texture`, and `craft-raster3d` depends on both. `craft-pbr` and
`craft-raster3d` do not depend on `craft-mesh-ops`: an app prepares a mesh with `craft-mesh-ops` and
passes the result to `craft-raster3d`. Each crate's first pull request proposes its API from the code it
moves.

### Mesh input

The public functions take plain arrays and slices (positions, normals, tangents with their handedness,
UVs and indices), as `craft-mesh` does. Tangents are optional, and a renderer given none does not apply
normal maps or anisotropy. Values are f64 at the file, document and outline boundary and f32 inside
`craft-pbr` and `craft-raster3d`. One documented conversion subtracts a scene origin before narrowing to
f32. No shared 3D math crate is added.

### Rules for every crate

- No UI and no file or network I/O. External files come through a callback, as `craft-mesh`'s
  `Resolver` does.
- `#![forbid(unsafe_code)]` and `[lints] workspace = true`.
- A `wasm32-unknown-unknown` build. Threads are behind a `parallel` feature.
- Deterministic output: no clock, and fixed seeds for sampled work.
- Limits are a parameter with documented defaults, as `craft_mesh::Limits` is. An input over a limit
  returns an error.
- A README with an Origin table and a Migration table that names at least two consumers.

### Material

`craft-pbr` defines the one material the shader reads. Its required part is the glTF 2.0
metallic-roughness core. Its optional part holds the OpenPBR lobes that TextureCraft already shades
(coat, sheen, anisotropy and transmission), and a renderer that does not implement a lobe ignores it. The
format crates do not depend on `craft-pbr`. Materials described by ambient, diffuse and specular colors
with a shininess value get one documented conversion to this material.

### sRGB

The new crates share one scalar sRGB encode and decode pair, in `craft-texture`. Its tests compare it
with `craft_cms::curve::srgb_trc()` (`crates/cms/src/curve.rs:24`).

### GGX geometry term

The crate has one reference formula. The move keeps EffectCraft's Smith-Schlick term, so EffectCraft's
tests pass unchanged. A numeric test records the difference from height-correlated Smith visibility over
a grid of roughness and angle values. A later pull request changes the reference formula only if the
apps whose output it changes accept the recorded difference. TextureCraft keeps its own BRDF until then.

### GPU

This RFC adds no shared wgpu pipeline. One is built on `craft-gpu`, which RFC-0022 decided and
storytold/craft-libs#663 added on 2026-11-19, in a later pull request that follows the CPU crates. Until
then each app keeps its own pipeline (EffectCraft `crates/gpu/src/adv3d.rs`, TextureCraft
`crates/render/src/renderer.rs`). The WGSL shading text moves with the CPU code it copies, and a
craft-libs test behind a feature compares the two on a software GPU driver.

## How the code moves

Each move is made in three kinds of pull request:

1. A craft-libs pull request adds the code with its tests.
2. One pull request per app switches the app to the crate and deletes the app's copy. Each moved test
   passes in the app or in the crate, and the pull request lists the moved tests.
3. A change to rendered output, such as MikkTSpace tangents in EffectCraft or EffectCraft's environment
   moving to the shared prefilter, is its own pull request with the measured difference.

The order follows the dependencies:

1. `craft-mesh-ops` is added, and TextureCraft and MaterialCraft switch to the one copy of their shared
   geometry code. `craft-texture`, which storytold/craft-libs#1216 added, gains the rest of the
   contents listed in "The crates", and MaterialCraft switches to it.
2. `craft-pbr` is added with the environment prefilter from their `ibl.rs`, and both apps switch to it.
   EffectCraft's BRDF moves into the same crate.
3. `craft-raster3d` is added, and EffectCraft switches its renderer to it.

Each app's switch is its maintainers' pull request. The order above is the order of the first consumers.
The other apps in Appendix A follow, each in its own switch pull request: PdfCraft to `craft-raster3d`
once it is on `main`, VectorCraft after storytold/craft-contrib#59 (RFC-0059) is decided, and DeckCraft,
PhotoCraft and CADCraft when their open pull requests continue.

A new crate adds no workflow. Its wasm32 check is a row in craft-libs `formats.yml`, added by a separate
pull request (house rule 11). An app that takes a shared crate moves every craft-libs dependency to one
revision. Every dependency comes from a storytold repository, because the `git-sources` job in
craft-common-actions `release.yml` fails on any other git source.

## VectorCraft

VectorCraft is a planned consumer of the four crates. How its 3D effects use them, with new effect
parameters and rendering tiers, is proposed in RFC-0059 (storytold/craft-contrib#59) and is not decided
here. VectorCraft's switch waits for that RFC.

## Defaults this RFC takes

1. **Names.** `craft-geom` is not used, because storytold/craft-libs#1132, storytold/craft-libs#138 and
   RFC-0030 use it for 2D code. `craft-gpu` is not used, because storytold/craft-libs#663 uses
   it.
2. **Geometry processing.** It goes in a new crate beside `craft-mesh`, because the `craft-mesh` README
   leaves normals, tangents and UVs to the apps. If the `craft-mesh` maintainers prefer it inside
   `craft-mesh`, the same code moves there behind features.
3. **The format program.** storytold/craft-libs#778 names no texture, shading or rasterizer crate, so
   the three new crates are proposed outside it. If a crate with the same contents is added under #778,
   this design uses that crate.
4. **Material content.** A comment on storytold/craft-libs#115 keeps 3D materials as app content
   (<https://github.com/storytold/craft-libs/issues/115#issuecomment-6171165239>). The crates hold code
   only. Material libraries and other assets stay in the apps.
5. **MikkTSpace.** The `mikktspace` 0.3.0 crate declares `MIT/Apache-2.0`. `MIT`, `Apache-2.0` and,
   since RFC-0027 (#41), `Zlib` are on the license allowlist in `standards/house-rules-rust.md`, so
   the crate is allowed whether or not its source files carry a Zlib notice.
6. **Consumers and RFC-0010.** RFC-0010 (#10), still open, sets when code may move into craft-libs,
   including its rule that a crate has two consuming apps and its rule that a crate names a responsible
   maintainer. The moves in this RFC follow whatever #10 decides. Each crate names at least two planned
   consumers in its README, but two crates have fewer than two consuming apps at the steps above:
   `craft-raster3d` has only EffectCraft at step 3, and only MaterialCraft depends on `craft-texture`
   on 2026-11-26. With all apps in scope, PdfCraft is a second consumer of `craft-raster3d` and
   TextureCraft a second consumer of `craft-texture`. This RFC does not name a maintainer for any of the
   crates.
7. **Other contributors' work.** This RFC changes no open pull request. When storytold/deckcraft#743,
   storytold/deckcraft#1380 or storytold/photocraft#8805 continues, it can use the shared crates. A CPU
   RENDER for storytold/cadcraft#1796 is a use of `craft-raster3d`. The `craft-pbr` material is the data
   that the per-surface material editing in storytold/effectcraft#2970 and storytold/effectcraft#2199
   would set.

## What this RFC decides

- The scope: every Craft app, with the four crates of "Scope" and no crate beyond them without its own
  pull request.
- The three new crates, the contents that `craft-texture` gains, and the names and boundaries of all
  four, as listed in "The crates".
- The source of each crate's code, and the reference copy of each piece in Appendix A.
- The mesh input, the material, the shared sRGB pair and the Smith-Schlick GGX default.

Per RFC-0000 this is a 3-day comment period change. The revision of 2026-11-26 widened the scope, so
the comment period restarts when it lands; the pull request's comment gives the end date.

## Appendix A: 3D code in every app

Read on 2026-11-26 with shallow clones of each repository's default branch. Commits: craft-libs
`f3d15cae`, effectcraft `8087064f`, texturecraft `d422c821`, materialcraft `e7ec9cb6`, pdfcraft
`09175f49`, vectorcraft `c8c4faa3`, deckcraft `74591aba`, cadcraft `b3bc4a76`, photocraft `4a4e3cff`,
designcraft `47e36bdd`, wordcraft `6c6ac698`, soundcraft `f07dc9e2`, filmcraft `a7dcaa22`, gridcraft
`4870243e`, lightcraft `3c6a9f2c`, figcraft `e1361cb1`, craft-anything `3e59a9c3`. Paths are in each app's
repository. "Reference" is the copy that becomes the crate's content.

| App | Code | Path | Crate | Reference |
|---|---|---|---|---|
| EffectCraft | Cook-Torrance BRDF, environment BRDF, PCF shadows, environment light | `crates/render/src/three_d/adv/shade.rs` | `craft-pbr` | Landed in craft-libs#2873; the app copy is deleted by EffectCraft's switch |
| EffectCraft | Default environment and scene data | `crates/render/src/three_d/adv/scene.rs` | `craft-pbr` | Landed in craft-libs#2873 |
| EffectCraft | WGSL copy of the shading | `crates/gpu/src/shaders/advanced3d.wgsl` | `craft-pbr` | Landed in craft-libs#2880 |
| EffectCraft | Depth-buffered rasterizer with shadow maps | `crates/render/src/three_d/adv/raster.rs` | `craft-raster3d` | EffectCraft's (craft-libs#2923) |
| EffectCraft | Normals, tangents, primitives, triangulation, extrusion | `crates/model/src/{lib,prim,triangulate,extrude}.rs` | `craft-mesh-ops` | EffectCraft's for extrusion and triangulation (craft-libs#2898); TextureCraft's for tangents |
| EffectCraft | Classic 3D plane compositor, cameras, Card Dance, 3D channels | `crates/render/src/three_d/{light,camera,compose}.rs`, `crates/effects/src/{card3d,channel3d}.rs`, `crates/raster/src/channels3d.rs` | none | Stays: built on the comp's layers and effect parameters |
| TextureCraft | Mesh, MikkTSpace tangents, BVH, UV unwrap, UV G-buffer, primitives, orbit camera | `crates/geom/src/{mesh,bvh,unwrap,gbuffer,primitives,camera}.rs` | `craft-mesh-ops` | TextureCraft's, with MaterialCraft's changes folded in |
| TextureCraft | CPU image-based lighting | `crates/render/src/ibl.rs` | `craft-pbr` | Landed in craft-libs#2850 |
| TextureCraft | Viewport and path-tracer shading with height-correlated Smith visibility | `crates/render/src/shaders/{mesh,trace}.wgsl`, `crates/render/src/trace.rs` | none yet | Stays until the GGX default question in "GGX geometry term" is settled |
| TextureCraft | Height to normal, ORM packing | `crates/compose/src/eval.rs`, `crates/formats/src/presets.rs` | `craft-texture` | Landed in `normal.rs` and `packing.rs` (craft-libs#1403) |
| MaterialCraft | Copy of TextureCraft's geometry code | `crates/geom/src/{mesh,bvh,unwrap,gbuffer,primitives,camera}.rs` | `craft-mesh-ops` | TextureCraft's; MaterialCraft's differences are separate pull requests |
| MaterialCraft | Image-based lighting | `crates/render/src/ibl.rs` | `craft-pbr` | Landed in craft-libs#2850 |
| MaterialCraft | CPU ray caster over a BVH with GGX shading | `crates/render/src/preview.rs` | consumer | Uses `craft-mesh-ops` for the BVH and `craft-pbr` for the BRDF |
| MaterialCraft | Image planes and sampling | was `crates/image` | `craft-texture` | Done: the app depends on `craft-texture` |
| PdfCraft | `Camera::fit`, work and size ceilings, depth-tested triangle fill | `crates/prc-render/src/{render,raster}.rs` | `craft-raster3d` | PdfCraft's for `Camera::fit` and ceilings (craft-libs#2958) |
| PdfCraft | Orbit navigation and vector helpers | `crates/prc-render/src/nav/orbit.rs`, `crates/prc-render/src/vec3.rs` | `craft-mesh-ops` | The camera of `craft-mesh-ops` (craft-libs#3664); PdfCraft's orbit moves if its tests show a difference |
| VectorCraft | 3D effects as shaded vector faces, painter's algorithm | `crates/three-d/src/{solid,shading,shadow,visibility,lights}.rs` | none | Stays: it draws vector gradients, not pixels; RFC-0059 decides its use of the crates |
| VectorCraft | Meshes of the effects' solids | `crates/three-d/src/mesh.rs` | `craft-mesh-ops` | A consumer of its mesh type; the extrusion code stays until RFC-0059 |
| VectorCraft | Depth-buffered fill of a folded packaging blank | `crates/three-d/src/fold/raster.rs` | `craft-raster3d` | To be checked when `craft-raster3d` lands: it fills panels with a depth buffer |
| DeckCraft | Transition layer mesh, 3-D format settings | `crates/anim/src/mesh.rs`, `crates/engine/src/cmd/three_d.rs` | none | Stays: 2D transition quads and XML settings; the open 3D pull requests use the crates |
| CADCraft | 3D polylines and helixes | `crates/engine/src/cmd/draw2/curves3d.rs` | none | No renderer on `main`; a CPU RENDER (storytold/cadcraft#1796) uses `craft-raster3d` |
| PhotoCraft | none on `main` | | | The 3D/UV workspace in storytold/photocraft#8805 uses the crates |
| DesignCraft, WordCraft, SoundCraft, FilmCraft, GridCraft, LightCraft, FigCraft, craft-anything | none found | | | |

Searched in each repository: mesh, normal and tangent functions, UV unwrapping, BVH, GGX and BRDF, IBL
and SH9, shadow maps and PCF, rasterizer and camera-fit types, mip and height-to-normal code, and WGSL
files. The hits in the other repositories were 2D code (curve tangents, `fit` functions for rectangles,
brush mip caches, BM3D denoising). Mesh file readers and writers (`craft-mesh`, `craft-usd`,
`craft-prc`) belong to the format program in storytold/craft-libs#778 and are not part of this RFC. The
360-degree equirectangular effects in EffectCraft (`crates/effects/src/vr.rs`) and FilmCraft
(`crates/render/src/vfx/immersive.rs`) are image effects and stay in the apps.
