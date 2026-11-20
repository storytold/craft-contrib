# RFC-0038: Roughness, Metallic and a rendered tier for VectorCraft's 3D effects

VectorCraft's 3D effects shade every face with one fixed material. This RFC adds a material to those
effects, and a second, rendered tier, using the shared crates that RFC-0032
(storytold/craft-contrib#50) proposes. RFC-0032 decides which shared crates exist. This RFC decides
how VectorCraft uses them.

Related: storytold/craft-contrib#50, storytold/vectorcraft#2560.

## Summary

VectorCraft's 3D effects gain a `material` parameter (Roughness, Metallic and texture maps), a `render`
parameter and an `environment` parameter. The default tier keeps the faces as vector paths and shades
them with `craft-pbr`. An opt-in tier renders the solid with `craft-raster3d` and draws the result as an
image. An effect that sets none of the three parameters draws as it does today.

## What exists

Read on 2026-11-15 at vectorcraft `5bd8acfc`.

- `crates/three-d` draws 3D effects as shaded vector faces. It depends only on `vectorcraft-geom` and
  `vectorcraft-pathops`.
- Colored lights paint a face with one linear gradient per color channel, fitted to three shaded values
  (`RgbGradient::triangle`, `crates/three-d/src/shading.rs:197`).
- Asset export writes `metallicFactor` 0 and `roughnessFactor` 1 for every mesh
  (`crates/engine/src/cmd/fileio/mesh3d.rs:171-172`).

storytold/vectorcraft#2560 asks for PBR texture sets and their import and export.

## The two tiers

1. **Vector (the default).** The faces stay vector paths. With a non-default material, each face is
   shaded with `craft-pbr` at three points. A face with a solid paint is painted with one linear
   gradient per color channel, fitted to the three values, as `RgbGradient::triangle` does today for
   colored lights. Its base color comes from the paint. A face with a gradient or pattern paint keeps
   that paint, and the shading is drawn over it as a separate layer, as VectorCraft does today.
   Roughness and Metallic apply, texture maps are ignored, and a base-color map contributes its
   average color. The effect's lights and the studio environment light the faces. The output prints
   and exports as paths.
2. **Rendered (opt-in per effect).** The solid becomes a triangle mesh with UVs, rendered by
   `craft-raster3d` with image-based lighting, shadow maps and every map, at the document's raster
   effects resolution. The app draws the result on the canvas and writes it to PDF, print and SVG output
   as an image.

## Parameters

Roughness and Metallic range from 0 to 1. A supplied value outside its range is rejected with an error,
and a default applies only to an omitted value. The default material is roughness 1, metallic 0 and no
maps. An effect that sets none of `material`, `render` and `environment` does not call `craft-pbr`. It
uses the existing shading in `crates/three-d`, unshaded mode and light softness included, so documents
made before these parameters draw as they do today, and a test checks that their output is unchanged.
An effect that sets `render` or `environment` and omits `material` uses the default material in the
tier that `render` selects.

## Where the code goes

`crates/three-d` keeps no craft-libs dependency: the shading call is in `crates/effects`, and the scene
for the rendered tier is built in `crates/render`. Asset export writes the material's roughness and
metallic factors. The vector tier needs `craft-texture` and `craft-pbr`. The rendered tier also needs
`craft-raster3d` and `craft-mesh-ops`.

## What this RFC decides

- The `material`, `render` and `environment` parameters and their ranges.
- The two tiers and the vector tier as the default.
- That an effect that sets none of the three parameters draws as it does today.

Work starts after this RFC and RFC-0032 are accepted, and each tier waits for the shared crates it
needs. Per RFC-0000 this is a 3-day comment period change.
