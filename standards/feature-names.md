# Names checked under RFC-0029

Set by [RFC-0040](../rfcs/0040-shared-names-list.md). The rule is
[RFC-0029](../rfcs/0029-other-companies-names.md). This list records how the
apps have applied it, so a name is checked once for the whole suite.

These are policy choices, not findings that a name infringes anything, and not
legal clearance.

**Before checking a name,** look for it here. A row with a decision and a check
record counts as already checked for any app that uses the name the same way.
**When an app reuses a feature,** use the name in "Our name" unless the app's
pull request says why not.

**Adding a row:** an ordinary pull request with a maintainer's approval. Link
the app's record (its pull request or `docs/feature-names.md`). A row that
disagrees with an earlier one goes to an owner.

## Not used as our feature's name

Test 1 is RFC-0029's "another company's name for its product, technology or
feature"; test 2 is "its owner claims it as a trademark for software".
"Proposed" means the app's pull request was still open on the date given.

| Name | Why | Our name | Record |
| --- | --- | --- | --- |
| Lumetri Color, Lumetri Scopes | Test 1: company feature names | Color Grading, Video Scopes (FilmCraft, EffectCraft) | RFC-0029 examples, 2026-11-15 |
| CC effects (CC Tiler) | Test 1: company effect family prefix | Descriptive names, e.g. Scaled Tiles (EffectCraft) | RFC-0029 examples, 2026-11-15 |
| SmartArt, WordArt | Test 1: company feature names | Diagram, Decorative Text (GridCraft; proposed in DeckCraft, WordCraft) | RFC-0029 examples, 2026-11-15 |
| Camera Raw, Camera Raw Filter | Test 1: company product name | RAW Develop, RAW Develop Filter (PhotoCraft) | RFC-0029 examples, 2026-11-15 |
| Upright | Test 1: company feature name | Perspective Correction (proposed, LightCraft) | RFC-0029 examples, 2026-11-15 |
| Express Tools | Test 1: company name for its tool set | Additional Tools (proposed, CADCraft) | RFC-0029 examples, 2026-11-15 |
| Dev Mode | Test 2: registered mark | Developer Mode (FigCraft) | RFC-0029 examples, 2026-11-15 |
| Beat Detective, QuickPunch | Test 1: company feature names | Beat Analysis, Punch Recording (SoundCraft) | RFC-0029 examples, 2026-11-15 |
| Adobe World-Ready Composer, Adobe Every-line Composer, Adobe Single-line Composer | Test 1: company prefix | Middle Eastern & South Asian Composer (PhotoCraft); others not chosen yet | [#43 inventory](https://github.com/storytold/craft-contrib/pull/43#issuecomment-6849272731) |
| Content-Aware Fill, Content-Aware Scale, Content-Aware Move, and "Content-Aware" as a mode in fill, patch, healing and crop | Test 1: one company's distinctive phrase | Not chosen yet | #43 inventory |
| Generative Fill, Generative Expand, Generate Image, Generate Background, Generate Similar, Generative Upscale, Generative Workspace | Test 1: company feature names | Not chosen yet | #43 inventory |
| Neural Filters, and its filter names (Smart Portrait, Skin Smoothing, Super Zoom, Depth Blur, Landscape Mixer, Harmonization, Style Transfer, Makeup Transfer, Photo Restoration) | Test 1: company feature names | Not chosen yet | #43 inventory |
| Firefly, Sensei | Test 1: company product and technology names | Not chosen yet | #43 inventory |
| Lightroom, Bridge, Mini Bridge, Adobe Stock, Adobe Fonts, Adobe Express, Creative Cloud, Creative Cloud Libraries, Cloud Documents | Test 1: company product names | Not chosen yet | #43 inventory |
| ExtendScript, UXP | Test 1: company technology names | Not chosen yet | #43 inventory |
| Adobe PDF Presets, Adobe Color Themes | Test 1: company prefix | Not chosen yet | #43 inventory |
| Photomerge | Test 1: coined, joined-up word | Not chosen yet | #43 inventory |
| Puppet Warp, Face-Aware Liquify | Test 1: one company's distinctive phrase | Not chosen yet | #43 inventory |
| Merge to HDR Pro, Contact Sheet II | Test 1: product-style suffix ("Pro", "II") | Not chosen yet | #43 inventory |

## Our wording that names another product

RFC-0029 allows factual compatibility statements in help, tooltips and import
messages, but not in our feature's label. These are our own wording.

| Text | Decision | Instead |
| --- | --- | --- |
| A menu label that names another product for what our export writes, e.g. "Paths to Illustrator" | Not used: it's our command's label | Name the file it writes |
| A marker on imported data that names the product it came from, e.g. "{set} (Photoshop)", "steps with no Photoshop equivalent" | Prefer the format | Name the file type the data came from or goes to |

## Not checked yet

Ordinary words that one company also uses for its version of the feature.
Each needs RFC-0029's check before an app keeps or adds it. Listing a name here
is a work list, not a finding.

| Name | Expected by |
| --- | --- |
| `Smart …` family: Smart Object, Smart Filters, Smart Sharpen, Smart Blur, Smart Guides, Smart Radius | PhotoCraft (family, RFC-0040 point 2) |
| `Magic …` family: Magic Wand, Magic Eraser | PhotoCraft (family, RFC-0040 point 2) |
| Select and Mask, Select Subject, Select Sky, Object Selection Tool, Object Finder, Quick Selection Tool, Quick Mask | PhotoCraft |
| Remove Tool, Spot Healing Brush, Healing Brush, Red Eye Tool | PhotoCraft |
| Magnetic Lasso, Background Eraser, Mixer Brush, History Brush, Art History Brush, Color Replacement Tool, Pattern Stamp | PhotoCraft |
| Curvature Pen, Freeform Pen, Frame Tool, Rotate View Tool | PhotoCraft |
| Perspective Warp, Vanishing Point, Adaptive Wide Angle, Liquify, Blur Gallery, Filter Gallery, Oil Paint, HDR Toning | PhotoCraft |
| Blend If, Layer Comps, Contextual Task Bar, Image Assets, Image Processor, Droplet, Script Events Manager | PhotoCraft |
| Quick Export, Save for Web, Copy CSS | PhotoCraft |
| Sky Replacement, Shake Reduction, Match Font, Adjustment Brush, Discover panel, Share for Review, Invite to Edit | PhotoCraft |
| Warp Stabilizer, Roto Brush, AudioSuite | Left to each app by RFC-0029 |

## Compatibility names that stay

Names of formats, colour spaces, file signatures and interchange data that we
read or write. They stay accurate where they refer to that thing (RFC-0029,
"What this covers").

| Name | Where |
| --- | --- |
| PSD, PSB, XMP, PSD-based format names (Photoshop PDF, Photoshop Raw, Photoshop EPS) | File format names |
| Adobe RGB (1998) | Colour space name |
| `%!PS-Adobe-` | EPS file header |
| Camera Raw `.xmp` settings files and their descriptor class names | Interchange data and document-owned names |
| The other application's name, when we look for it installed to edit embedded content, or import settings from its settings folder | Factual compatibility statement |
| Content Credentials | C2PA standard, once an app reads or writes it |
