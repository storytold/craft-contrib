# RFC-NNNN: Permitted Use of Model Parameters Fitted to Proprietary App Outputs

Updates: [RFC-0072](0072-shared-agents-md.md)

Standards: [standards/clean-room.md](../standards/clean-room.md)

## Summary

This RFC settles item 3, "Measurements", of `standards/clean-room.md` § "Observing a proprietary app".

It establishes a clean-room protocol allowing the project to derive parameters for our own open mathematical models (such as 3×3 colour transformation matrices) by comparing contributor-owned raw files against neutral exports rendered by a proprietary vendor application. The protocol enforces strict separation between an Observer agent (who operates the application as a black box) and an Implementer agent (who performs numerical fitting on decoupled pixel data), ensures no proprietary code, assets, profiles, or LUTs are copied, and specifies what may be committed (fitted numbers and provenance records) and what must remain local (rendered exports).

## Motivation

Several digital camera bodies—notably the Canon EOS 6D and the Canon EOS Rebel T100 / 2000D / 3000D / 4000D—lack published spectral sensitivities in open datasets like Academy Software Foundation's `rawtoaces-data`. Without measured sensitivities or physical ColorChecker calibration captures, LightCraft cannot construct accurate DNG colour matrices (`ColorMatrix1/2`, `ForwardMatrix1/2`), causing raw captures from these cameras to render with severe colour distortion or fall back to an uncalibrated default.

Currently, `standards/clean-room.md` § "Observing a proprietary app" freezes all such work under item 3:

> **3. Measurements.** Numbers measured from the app's output, such as exports compared with each other or parameters of our own model fitted to them, are not yet covered by these rules and need their own decision. Until then, you MUST NOT start work that uses them; existing work stays as it is.

Vendors supply raw processing software (such as Canon Digital Photo Professional / DPP) capable of rendering raw files shot on their hardware into standard, uncompressed output (e.g. 16-bit linear or sRGB TIFFs). Treating the vendor application strictly as an external black box provides an empirical baseline to fit open, general mathematical models without inspecting, decompiling, or copying any internal proprietary assets.

A clear, audited rule is needed so Craft applications can support cameras missing from open spectral libraries while preserving strict clean-room boundaries.

## Proposal

We propose replacing item 3 of `standards/clean-room.md` § "Observing a proprietary app" with the following standard protocol:

### 1. Two-Role Clean-Room Boundary
The strict separation between Observer and Implementer (RFC-0072) is preserved and enforced:
- **Observer Agent:** Operates the vendor application strictly via clean black-box automation or documented GUI steps. The observer:
  - MUST NOT inspect memory, run debuggers, trace syscalls, decompile, or disassemble application binaries.
  - MUST NOT inspect or extract internal resources, bundled ICC/DCP profiles, proprietary LUTs, recipe files, or tone curves.
  - Generates exports exclusively from permitted inputs using documented, neutral settings (e.g. Neutral/Faithful picture style, disabled noise reduction, disabled lens corrections, as-shot or fixed daylight white balance).
  - Produces a behavior specification document recording exact application version, platform, input file hashes, and all export parameters used.
- **Implementer Agent:** Writes code and performs mathematical modeling. The implementer:
  - MUST NOT operate the proprietary application or access its installation files.
  - Receives only decoupled input/output pixel pairs or patch statistics and the observer's behavior specification.
  - Fits parameters for *our own* mathematical model.

### 2. Permitted Inputs
- The proprietary application MUST only be run with raw image inputs that a contributor owns, has shot personally, or that the project generates.
- Third-party copyrighted photographs, proprietary vendor sample files, and images containing personal data or non-consenting individuals MUST NOT be used.

### 3. Model Independence & Asset Boundaries
- The parameters fitted MUST belong strictly to our own documented, open mathematical model (for example, standard 3×3 colour matrices mapping camera RGB to CIE XYZ under D50/D65 illuminants).
- The fitting process MUST NOT extract, approximate, or reconstruct proprietary multi-dimensional LUTs, proprietary profile formats, or vendor-specific film simulation recipes.
- The output of the vendor tool serves solely as an external photometric reference, analogous to photographing a physical reflectance chart.

### 4. Commit Boundaries & Provenance
- **What MAY be committed:**
  - The resulting fitted scalar coefficients (e.g. 3×3 matrix arrays in Rust tables).
  - A provenance record in the repository attribution file (`ATTRIBUTION.md`) citing the camera model, input raw SHA-256 hashes, vendor application version, documented export settings, and fitting script/method.
  - The fitting script, provided it contains only standard optimization mathematics and no proprietary code or data.
- **What MUST NOT be committed:**
  - Rendered export images (TIFFs, JPEGs), intermediate pixel dumps, diff masks, and application screenshots MUST NOT be committed to the repository, included in test fixtures, or uploaded to PRs or issue trackers.
  - All intermediate artifacts MUST remain strictly in the gitignored `plan/` directory or temporary local scratch storage.

## Changes to standards

In `standards/clean-room.md`, under `## Observing a proprietary app`, replace item 3 with:

```markdown
3. **Measurements and fitted models.** You MAY fit parameters of our own documented mathematical model (such as a 3×3 colour matrix or a global tone curve) to pairs of inputs and exported outputs from a proprietary app, provided:
   - the inputs are files a contributor owns or we generate;
   - the app is operated as a black box by an Observer agent with settings recorded in a behaviour specification;
   - no code, binary, shader, profile, LUT or asset inside the app's install is inspected or copied;
   - an Implementer agent who has not operated the app fits our own model to the decoupled data;
   - the fitted numbers and their provenance are recorded in `ATTRIBUTION.md`;
   - the exported images and intermediate captures remain in the gitignored `plan/` folder and are never committed or published.
```

In `standards/app-agents.md`, update the summary reference to clean-room observation accordingly.

## Alternatives Considered

1. **Physical ColorChecker captures under calibrated illuminants:**
   - *Approach:* Photographing physical ColorChecker charts under calibrated 5000 K / 2856 K lighting with every unsupported camera body, as done for the EOS D60 (`clean-room/crw/10-chart-calibration.md`).
   - *Why it falls short:* Requires physical possession of every camera body, calibrated lighting rigs, and physical reference targets. For out-of-production bodies (EOS 6D) or low-cost regional bodies (Rebel T100 / 4000D), contributors often have raw files but lack physical measurement laboratories.
2. **Waiting for open spectral sensitivity databases (`rawtoaces-data`):**
   - *Approach:* Waiting for Academy Software Foundation or academic labs to publish laboratory monochromator measurements for missing cameras.
   - *Why it falls short:* `rawtoaces-data` has been stagnant on older consumer bodies; recent cameras and budget models are rarely measured in academic color science labs.
3. **Embedded-JPEG calibration fallback:**
   - *Approach:* Fitting matrices by comparing raw sensor values to in-camera embedded preview JPEGs.
   - *Why it falls short:* In-camera JPEGs have non-linear tone curves, dynamic range optimization, sharpening, and proprietary color matrix twists baked into 8-bit YCbCr pixels, introducing significant error compared to a 16-bit linear or neutral export from an official raw developer.
4. **Reading vendor SDKs or decompiling binaries:**
   - *Approach:* Decompiling vendor raw processing libraries or reading leaked SDK header files.
   - *Why it falls short:* Directly violates Rule 23 of `AGENTS.md` and foundational clean-room principles.

## Open Questions

1. **Vendor End-User License Agreements (EULAs):**
   - Proprietary raw converters typically ship with click-through EULAs containing boilerplate clauses regarding reverse engineering or benchmarking.
   - *Note:* Under our clean-room rules, an agent MUST NOT read, inspect, or decompile anything inside the app's install or bundle. Consequently, the agent has not inspected the installer's EULA text.
   - *Legal consideration:* The human owner or maintainer must verify that running the vendor application locally on contributor-owned files to export images does not violate applicable local end-user laws, fair use, or interoperability exemptions.
2. **Reproducibility verification:**
   - Since intermediate exported TIFFs cannot be committed to the repository, how can independent reviewers verify the numerical optimization?
   - *Proposed approach:* Require the fitting script to accept standard directory layouts and verify checksums of input raws, so any contributor with access to the black-box application can re-run the optimization and confirm matrix convergence within numerical tolerances.
