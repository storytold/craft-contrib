# Names

The rules for names that belong to other companies, from [RFC-0029](../rfcs/0029-other-companies-names.md), in short form. Link this file from a repository's `AGENTS.md`. A wording fix or a new row in the names list is an ordinary pull request. Changing a rule needs an RFC that updates RFC-0029.

This is a naming policy, not legal advice or trademark clearance.

## Rules

1. **Our feature uses our own name.** Don't name it with another company's product, technology or distinctive feature name, or with a name its owner claims as a software trademark (on its trademark list, marked ™ or ® in its docs, or found by the check below). Use a descriptive name of our own.
2. **Scope.** This covers what users see: labels, menus, dialogs, messages, tooltips, help, READMEs, store and release text, and every translation. It doesn't cover command ids, preference keys, stored values, code, comments, tests or internal comparison documents. No other companies' logos or icons, and no promotional comparisons ("Figma-class").
3. **These stay accurate:** file formats, languages and APIs we support (`Open PSD`); names that belong to the user's documents, and interchange names, also in files we export; protocols and plug-in formats once the integration works (until then, hide the entry or show it disabled); factual compatibility statements ("opens PSD files") in help, tooltips and import messages, but never in our feature's label; legal notices and attribution.
4. **Check every new or changed name**, including a replacement, unless it is already in the list below and used the same way:
   - the owner's trademark list, and ™ or ® in its documentation;
   - [USPTO](https://tmsearch.uspto.gov/), [EUIPO eSearch](https://euipo.europa.eu/eSearch/) or [TMview](https://www.tmdn.org/tmview/), and the [WIPO Global Brand Database](https://branddb.wipo.int/), for the name and obvious variants;
   - to call a name common, three unrelated products that use it descriptively for their own feature.

   Record the date, sources, findings and decision in the pull request and in the app's `docs/feature-names.md`. Finding nothing doesn't mean the name is free.
5. **Unclear means no.** If the check doesn't settle it, don't ship the name. Use our own, or ask a Craft owner on the pull request. Contributors aren't expected to make legal calls.
6. **Change labels only.** Keep ids, preference keys and stored values. Keep the old label as a search alias that isn't displayed. Translate the new label.

## Names list

Policy choices, not findings that a name infringes anything. "Proposed" means the app's pull request was still open on 2026-11-15.

### Not used as our feature's name

| Name | Why | Our name |
| --- | --- | --- |
| Lumetri Color, Lumetri Scopes | Adobe feature names | Color Grading, Video Scopes (FilmCraft, EffectCraft) |
| CC effects (CC Tiler) | Cycore's effect family prefix | Descriptive names, e.g. Scaled Tiles (EffectCraft) |
| SmartArt, WordArt | Microsoft feature names | Diagram, Decorative Text (GridCraft; proposed in DeckCraft, WordCraft) |
| Camera Raw, Camera Raw Filter | Adobe product name | RAW Develop, RAW Develop Filter (PhotoCraft) |
| Upright | Adobe feature name | Perspective Correction (proposed, LightCraft) |
| Express Tools | AutoCAD's name for its tool set | Additional Tools (proposed, CADCraft) |
| Dev Mode | Figma's registered mark | Developer Mode (FigCraft) |
| Beat Detective, QuickPunch | Avid feature names | Beat Analysis, Punch Recording (SoundCraft) |
| Content-Aware Fill, Content-Aware Scale, Content-Aware Move, and "Content-Aware" as a fill, patch, healing or crop mode | One company's distinctive phrase | Not chosen yet |
| Generative Fill, Generative Expand, Generate Image, Generate Background, Generate Similar, Generative Upscale | Adobe feature names | Not chosen yet |
| Neural Filters, and its filter names (Smart Portrait, Super Zoom, Landscape Mixer, Makeup Transfer and others) | Adobe feature names | Not chosen yet |
| Firefly, Sensei, Lightroom, Bridge, Adobe Stock, Adobe Fonts, Creative Cloud, ExtendScript, UXP | Adobe product and technology names | Not chosen yet |
| Photomerge, Puppet Warp, Face-Aware Liquify | Coined word; one company's distinctive phrases | Not chosen yet |
| Merge to HDR Pro, Contact Sheet II | Product-style suffix ("Pro", "II") | Not chosen yet |
| Adobe World-Ready Composer, Adobe Every-line Composer, Adobe Single-line Composer | Company prefix | Middle Eastern & South Asian Composer (PhotoCraft); others not chosen yet |
| Warp Stabilizer, Roto Brush, AudioSuite | Left to each app by RFC-0029 | Each app decides |

### Our wording that names another product

| Text | Instead |
| --- | --- |
| A label naming the product our export targets ("Paths to Illustrator") | Name the file it writes |
| Markers on imported data naming the product ("{set} (Photoshop)", "steps with no Photoshop equivalent") | Name the file type |

### Check before use

Ordinary words that one company also uses for its version. A family (`Smart …`, `Magic …`) can be checked once for all its members.

Smart Object, Smart Filters, Smart Sharpen, Smart Blur, Smart Guides, Magic Wand, Magic Eraser, Select and Mask, Select Subject, Select Sky, Spot Healing Brush, Quick Selection Tool, Liquify, Vanishing Point, Perspective Warp, Adaptive Wide Angle, Blur Gallery, Layer Comps, Quick Export, Save for Web.

### Stay as compatibility names

PSD, PSB, XMP, DOCX, MusicXML; Adobe RGB (1998); the `%!PS-Adobe-` EPS header; Photoshop PDF, Photoshop Raw and Photoshop EPS as format names; Camera Raw `.xmp` settings and their descriptor names; `Lumetri Color` in FCPXML; `Camera Raw Filter` as a step in a recorded action.
