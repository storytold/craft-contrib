# RFC-0032: Shared crates for PBR materials and 3D rendering

This RFC revises the 2026-10-27 design by addibble
(<https://gist.github.com/addibble/0895ec701abaa70ac8f4bf6154750317>), which proposed moving
EffectCraft's physically based renderer into new craft-libs crates. Since that design, TextureCraft and
MaterialCraft have added their own PBR code, craft-libs has added mesh, MaterialX, OpenEXR and Radiance
HDR crates, and RFC-0022 has decided the GPU layer. This revision plans the move with those changes.

Related: storytold/craft-libs#778, storytold/craft-libs#115, storytold/vectorcraft#2560,
storytold/effectcraft#2970, storytold/effectcraft#2199, storytold/deckcraft#743,
storytold/deckcraft#1380, storytold/photocraft#8805, storytold/cadcraft#1796.

## Summary

Four new craft-libs crates hold the 3D code that several apps now carry separately: `craft-mesh-ops`,
`craft-texture`, `craft-pbr` and `craft-raster3d`. Each crate is filled from the app whose code already
has tests for that part. The crates add no importer or decoder. They read the output of `craft-mesh`,
`craft-materialx`, `craft-hdr`, `craft-exr` and `craft-raster-codecs`. VectorCraft uses them to add
Roughness and Metallic to its 3D effects, in two tiers.

## What exists

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
- **VectorCraft.** `crates/three-d` draws 3D effects as shaded vector faces. Asset export writes
  `metallicFactor` 0 and `roughnessFactor` 1 for every mesh
  (`crates/engine/src/cmd/fileio/mesh3d.rs:171-172`).
- **craft-libs.** `craft-mesh` reads mesh files into one `Scene`, and its README leaves computing
  normals, tangents and UVs to the apps. `craft-materialx`, `craft-exr`, `craft-hdr` and
  `craft-raster-codecs` exist. No crate holds textures with sampling, a BRDF, image-based lighting or a
  software 3D rasterizer.

Open pull requests add further renderers: storytold/deckcraft#743, storytold/deckcraft#1380 and
storytold/photocraft#8805. storytold/cadcraft#1796 asks for a CPU renderer for RENDER.

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

No shared wgpu pipeline is added until storytold/craft-libs#663 (RFC-0022) merges. The WGSL shading text
moves with the CPU code it copies, and a craft-libs test behind a feature compares the two on a software
GPU driver.

## How the code moves

Each move is made in three kinds of pull request:

1. A craft-libs pull request adds the code with its tests.
2. One pull request per app switches the app to the crate and deletes the app's copy. Each moved test
   passes in the app or in the crate, and the pull request lists the moved tests.
3. A change to rendered output, such as MikkTSpace tangents in EffectCraft or EffectCraft's environment
   moving to the shared prefilter, is its own pull request with the measured difference.

The order follows the dependencies:

1. `craft-mesh-ops` and `craft-texture` are added, and TextureCraft and MaterialCraft switch to the one
   copy of their shared geometry code.
2. `craft-pbr` is added with the environment prefilter from their `ibl.rs`, and both apps switch to it.
   EffectCraft's BRDF moves into the same crate.
3. `craft-raster3d` is added, and EffectCraft switches its renderer to it.

A new crate adds no workflow. Its wasm32 check is a row in craft-libs `formats.yml`, added by a separate
pull request (house rule 11). An app that takes a shared crate moves every craft-libs dependency to one
revision. Every dependency comes from a storytold repository, because the `git-sources` job in
craft-common-actions `release.yml` fails on any other git source.

## VectorCraft

VectorCraft's 3D effects gain a `material` parameter (Roughness, Metallic and texture maps), a `render`
parameter and an `environment` parameter, in two tiers:

1. **Vector (the default).** The faces stay vector paths. With a non-default material, each face is
   shaded with `craft-pbr` at three points. A face with a solid paint is painted with one linear
   gradient per color channel, fitted to the three values, as `RgbGradient::triangle` does today for
   colored lights (`crates/three-d/src/shading.rs:197`). Its base color comes from the paint. A face
   with a gradient or pattern paint keeps that paint, and the shading is drawn over it as a separate
   layer, as VectorCraft does today. Roughness and Metallic apply, texture maps are ignored, and a
   base-color map contributes its average color. The effect's lights and the studio environment light
   the faces. The output prints and exports as paths.
2. **Rendered (opt-in per effect).** The solid becomes a triangle mesh with UVs, rendered by
   `craft-raster3d` with image-based lighting, shadow maps and every map, at the document's raster
   effects resolution. The app draws the result on the canvas and writes it to PDF, print and SVG output
   as an image.

Roughness and Metallic range from 0 to 1. A supplied value outside its range is rejected with an error,
and a default applies only to an omitted value. The default material (roughness 1, metallic 0, no maps,
vector tier) does not call `craft-pbr`. It uses the existing shading in `crates/three-d`, unshaded mode
and light softness included, so documents without a `material` parameter draw as they do today, and a
test checks that their output is unchanged. `crates/three-d` keeps no craft-libs dependency: the
shading call is in `crates/effects`, and the scene for the rendered tier is built in `crates/render`.
Asset export writes the material's roughness and metallic factors. The vector tier needs
`craft-texture` and `craft-pbr`. The rendered tier also needs `craft-raster3d` and `craft-mesh-ops`.

## Defaults this RFC takes

1. **Names.** `craft-geom` is not used, because storytold/craft-libs#1132, storytold/craft-libs#138 and
   RFC-0030 (#46) use it for 2D code. `craft-gpu` is not used, because storytold/craft-libs#663 uses
   it.
2. **Geometry processing.** It goes in a new crate beside `craft-mesh`, because the `craft-mesh` README
   leaves normals, tangents and UVs to the apps. If the `craft-mesh` maintainers prefer it inside
   `craft-mesh`, the same code moves there behind features.
3. **The format program.** storytold/craft-libs#778 names no texture, shading or rasterizer crate, so
   the four crates are proposed outside it. If a crate with the same contents is added under #778, this
   design uses that crate.
4. **Material content.** A comment on storytold/craft-libs#115 keeps 3D materials as app content
   (<https://github.com/storytold/craft-libs/issues/115#issuecomment-6171165239>). The crates hold code
   only. Material libraries and other assets stay in the apps.
5. **MikkTSpace.** The `mikktspace` 0.3.0 crate declares `MIT/Apache-2.0`. If its source files carry a
   Zlib notice, its use waits for RFC-0027 (#41), because the license allowlist does not include Zlib.
6. **Consumers.** Each crate names at least two consuming apps in its README, which meets the craft-libs
   rule that a crate is useful to more than one app.
7. **Other contributors' work.** This RFC changes no open pull request. When storytold/deckcraft#743,
   storytold/deckcraft#1380 or storytold/photocraft#8805 continues, it can use the shared crates. A CPU
   RENDER for storytold/cadcraft#1796 is a use of `craft-raster3d`. The `craft-pbr` material is the data
   that the per-surface material editing in storytold/effectcraft#2970 and storytold/effectcraft#2199
   would set.

## What this RFC decides

- The four crates, their names and their boundaries, as listed in "The crates".
- The source of each crate's code.
- The mesh input, the material, the shared sRGB pair and the Smith-Schlick GGX default.
- VectorCraft's two tiers.

Work in TextureCraft and MaterialCraft starts after this RFC is accepted, or after a maintainer of each
of the two apps agrees in its review. Per RFC-0000 this is a 3-day comment period change.
