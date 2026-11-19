# Names

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as in RFC 2119 and RFC 8174.

The rules for names that belong to other companies, from [RFC-0029](../rfcs/0029-other-companies-names.md), in short form. They apply in every Craft repository; the shared `AGENTS.md` links this file. A wording fix or a new row in the names list is an ordinary pull request. Changing a rule needs an RFC that updates RFC-0029.

This is a naming policy, not legal advice or trademark clearance.

## Rules

1. **Our feature uses our own name.** Feature names MUST NOT use another company's trademarked product or feature names: a product, technology or distinctive feature name, or a name its owner claims as a software trademark (on its trademark list, marked ™ or ® in its docs, or found by the check below). Describe the feature in our own words. Ordinary descriptive words aren't excluded just because one company used them first. A name that is neither of these is allowed: usually one that describes what the feature does in ordinary words, the way several unrelated products describe the same thing (`Layers`, `Crop`, `Find and Replace`).
2. **Scope.** This covers what users see: labels, menus, dialogs, messages, tooltips, help, READMEs, store and release text, and every translation. It doesn't cover command ids, preference keys, stored values, code, comments, tests or internal comparison documents. You SHOULD NOT use other companies' logos or icons, or promotional comparisons ("Figma-class").
3. **These MAY stay accurate:** file formats, languages and APIs we support (`Open PSD`); names that belong to the user's documents, and interchange names, also in files we export; protocols and plug-in formats once the integration works (until then, hide the entry or show it disabled); factual compatibility statements ("opens PSD files") in help, tooltips and import messages, but not in our feature's label; legal notices and attribution.
4. **Check every new or changed name** (SHOULD), including a replacement, unless it is already in the list below and used the same way:
   - the owner's trademark list, and ™ or ® in its documentation;
   - [USPTO](https://tmsearch.uspto.gov/), [EUIPO eSearch](https://euipo.europa.eu/eSearch/) or [TMview](https://www.tmdn.org/tmview/), and the [WIPO Global Brand Database](https://branddb.wipo.int/), for the name and obvious variants;
   - to call a name common, three unrelated products that use it descriptively for their own feature.

   You SHOULD record the date, sources, findings and decision in the pull request and in the app's existing feature-naming document (`docs/feature-names.md` if it has none). Finding nothing doesn't mean the name is free.
5. **Unclear means no.** If the check doesn't settle it, you SHOULD NOT ship the name. Use our own, or ask a Craft owner on the pull request. Owners decide; contributors aren't expected to make legal calls.
6. **Renaming.** A rename SHOULD change labels only. It SHOULD keep ids, preference keys and stored values, and SHOULD keep the old label as a search alias that isn't displayed. The new label SHOULD be translated.

## Names list

Policy choices, not findings that a name infringes anything. A name marked *US-checked* passed two parts of rule 4: the owner's trademark list and a US register search, with descriptive uses by other products. The app pull request that first uses the name runs the WIPO Global Brand Database search and updates the row.

### Not used as our feature's name

| Name | Why | Our name |
| --- | --- | --- |
| Lumetri Color, Lumetri Scopes | Adobe feature names | Color Grading, Video Scopes (FilmCraft, EffectCraft) |
| CC effects (CC Tiler) | Cycore's effect family prefix | Descriptive names, e.g. Scaled Tiles (EffectCraft) |
| SmartArt, WordArt | Microsoft feature names | Diagram, Decorative Text (GridCraft, DeckCraft, WordCraft) |
| Camera Raw, Camera Raw Filter | Adobe product name | RAW Develop, RAW Develop Filter (PhotoCraft) |
| Upright | Adobe feature name | Perspective Correction (LightCraft) |
| Express Tools | AutoCAD's name for its tool set | Additional Tools (CADCraft) |
| Dev Mode | Figma's registered mark | Developer Mode (FigCraft) |
| Beat Detective, QuickPunch | Avid feature names | Beat Analysis, Punch Recording (SoundCraft) |
| Content-Aware Fill | One company's distinctive phrase | *US-checked:* Fill from Surroundings |
| Content-Aware Scale | One company's distinctive phrase | *US-checked:* Subject-Preserving Scale |
| Content-Aware Move Tool | One company's distinctive phrase | *US-checked:* Seamless Move Tool |
| "Content-Aware" as a mode in fill, patch, healing or crop | One company's distinctive phrase | *US-checked:* Surroundings (crop: Fill from Surroundings) |
| Generative Fill, Generative Expand, Generate Image, Generate Background, Generate Similar, Generative Upscale | Adobe feature names | *US-checked:* AI Fill, AI Expand, AI Image, AI Background, More Variations, AI Upscale |
| Neural Filters, and its filter names Smart Portrait, Super Zoom, Landscape Mixer, Makeup Transfer | Adobe feature names | *US-checked:* AI Filters; Portrait Adjust, AI Enlarge, Scene Restyle, Makeup Match. Its other filter names follow the same rule |
| Firefly, Sensei | Adobe product and technology names | AI, the suite's shared AI layer; never a model or vendor name as a label |
| Lightroom, Bridge, Adobe Stock, Adobe Fonts, Creative Cloud | Adobe product names | *US-checked:* Photo Library, File Browser, Stock Images, Online Fonts, Shared Libraries. Not Cloud Library: `CLOUDLIBRARY` is a registered US mark |
| ExtendScript, UXP | Adobe technology names | *US-checked:* Scripts, Plug-ins |
| Photomerge | Coined, joined-up word | *US-checked:* Panorama |
| Puppet Warp | One company's distinctive phrase | No checked name yet. Not Pin Warp, another company's filter name, and not Deform Pins, Adobe's wording. The first app to use one checks it |
| Face-Aware Liquify | One company's distinctive phrase | *US-checked:* Face Reshape |
| Merge to HDR Pro, Contact Sheet II | Product-style suffix ("Pro", "II") | *US-checked:* Merge to HDR, Contact Sheet |
| Adobe World-Ready Composer, Adobe Every-line Composer, Adobe Single-line Composer | Company prefix | Middle Eastern & South Asian Composer (PhotoCraft); *US-checked:* Paragraph Line Breaking, Single-Line Line Breaking. Not Paragraph Composer or Line Composer: only Adobe was found using them |
| Warp Stabilizer, Roto Brush, AudioSuite | Left to each app by RFC-0029 | Each app decides |

### Our wording that names another product

| Text | Instead |
| --- | --- |
| A label naming the product our export targets ("Paths to Illustrator") | Name the file it writes |
| Markers on imported data naming the product ("{set} (Photoshop)", "steps with no Photoshop equivalent") | Name the file type |

### Not checked yet

These are plain words, so they may well be fine to use. But one company also uses each of them as the name of its own version of the feature, and plain words can still be registered (`Dev Mode` was). Nobody has run the rule 4 check on them yet. The first app pull request that keeps or adds one runs the check and moves the name into a table above. A family (`Smart …`, `Magic …`) can be checked once for all its members.

Smart Object, Smart Filters, Smart Sharpen, Smart Blur, Smart Guides, Magic Wand, Magic Eraser, Select and Mask, Select Subject, Select Sky, Spot Healing Brush, Quick Selection Tool, Liquify, Vanishing Point, Perspective Warp, Adaptive Wide Angle, Blur Gallery, Layer Comps, Quick Export, Save for Web.

### Stay as compatibility names

PSD, PSB, XMP, DOCX, MusicXML; Adobe RGB (1998); the `%!PS-Adobe-` EPS header; Photoshop PDF, Photoshop Raw and Photoshop EPS as format names; Camera Raw `.xmp` settings and their descriptor names; `Lumetri Color` in FCPXML; `Camera Raw Filter` as a step in a recorded action.
