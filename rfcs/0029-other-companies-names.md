# RFC-0029: Using other companies' names

This RFC sets one rule for when the Craft apps use names that belong to other companies, so the answer is the same in every app and easy to check.

This is a conservative naming policy, not legal advice or trademark clearance. It keeps our own features under our own names, and sends unclear cases to an owner.

## Rule

Don't use a name for our own feature when either is true:

1. **It is another company's name for its product, technology or feature.** That includes a distinctive feature name a company uses for its own version of something, even if it isn't registered: users read it as that company's feature. Ordinary descriptive words aren't excluded just because one company used them first.
2. **Its owner claims it as a trademark for software.** That means it is on the owner's published trademark list, the owner marks it ™ or ® in its own documentation, or a relevant application or registration turns up in the check below. A listing for unrelated goods doesn't count.

Use our own descriptive name instead.

A maintainer may also choose a different name for clarity, for example when a name doesn't describe our feature accurately or is inconsistent within the suite. The pull request says why.

## What this covers

The rule covers what users see as the name of our feature: labels, menus, dialogs, messages, tooltips, help, READMEs, store and release text, and every translation of them. Don't use another company's name as a promotional comparison in product copy (for example, "Figma-class"); factual compatibility statements below are allowed.

These stay accurate wherever they appear, including in translations:

- names of file formats, programming languages and APIs the app supports (`Export MusicXML`, `Open PSD`);
- names that belong to the user's documents, and stable interoperability names, including in files we export;
- protocols and plug-in formats once the integration works, following the owner's guidelines. Until then, hide the entry or show it disabled without implying support;
- factual compatibility statements ("opens PSD files") in help, documentation, tooltips and import messages, but not in product names, taglines or our feature's label;
- legal notices and attribution.

It doesn't cover command ids, preference keys, stored values, code, comments, tests, or internal documents that compare us with another product. That exception covers the comparison, not claims about our own features in the same document. Don't use other companies' logos or icons.

## Common feature names

A name that meets neither test is fine. That is usually a name that describes what the feature does in ordinary words, the way several unrelated products describe the same thing (`Layers`, `Crop`, `Find and Replace`).

It is usually not fine when it is a coined or joined-up word (`SmartArt`, `QuickPunch`), carries a company or product prefix (`Lumetri`, `CC`), or is one company's distinctive phrase for its version of a feature (`Warp Stabilizer`).

Ordinary words can still be claimed: Figma registered DEV MODE in the US, so `Dev Mode` fails test 2 although both words are common. That is why a name gets checked, not just judged by how it looks.

## Checking a name

When a pull request introduces or changes a feature name, including a replacement, it records a short check. A name already checked and used the same way doesn't need a new one.

1. **Owner's list.** If the name comes from another company, search its published trademark list for the name and its spellings. These lists aren't complete.
2. **Owner's documentation.** Look for the name with ™ or ®.
3. **Registers.** Search the name and obvious variants in the [USPTO](https://tmsearch.uspto.gov/), either [EUIPO eSearch](https://euipo.europa.eu/eSearch/) or [TMview](https://www.tmdn.org/tmview/), and the [WIPO Global Brand Database](https://branddb.wipo.int/). Note any application or registration that looks relevant, with its status and goods. Classes 9 and 42 are useful starting points, not a boundary.
4. **Common use.** If you say a name is common, link three unrelated products that use it descriptively for their own equivalent feature, not as a brand or a compatibility reference.

Record the date, the sources searched, what turned up and the decision, in the pull request and the app's existing feature-naming document (`docs/feature-names.md` if it has none). A search that finds nothing means nothing turned up in those sources, not that the name is free to use. The reviewing maintainer checks that the record supports the decision.

A known claim or complaint about a name we use gets a new check straight away.

## When it's unclear

If the check doesn't settle it, for example a relevant registration or a match you can't judge, don't ship the name. Use our own and say so in the pull request, or ask a Craft owner on it. Owners decide, and can get legal advice. Maintainers and contributors aren't expected to make legal judgements.

## Examples

These show how the rule applies. They are policy choices, not findings that a name infringes anything. "Proposed" means the app's pull request is still open.

| Name | Use it as our feature's name? | Why | Our name |
| --- | --- | --- | --- |
| Lumetri Color, Lumetri Scopes | No | Adobe feature names | Color Grading, Video Scopes (FilmCraft, EffectCraft) |
| CC effects (CC Tiler) | No | Cycore's effect family prefix | Descriptive names, e.g. Scaled Tiles (EffectCraft) |
| SmartArt, WordArt | No | Microsoft feature names | Diagram, Decorative Text (GridCraft; proposed in DeckCraft, WordCraft) |
| Camera Raw | No | Adobe product name | RAW Develop (PhotoCraft) |
| Upright | No | Adobe feature name | Perspective Correction (proposed, LightCraft) |
| Express Tools | No | AutoCAD's name for its tool set | Additional Tools (proposed, CADCraft) |
| Dev Mode | No | Figma's registered mark | Developer Mode (FigCraft) |
| Beat Detective, QuickPunch | No | Avid feature names | Beat Analysis, Punch Recording (SoundCraft) |
| PSD, DOCX, MusicXML, XMP | Only for the format | File formats we support | — |
| `Lumetri Color` in FCPXML, `Camera Raw Filter` in a Photoshop action | Only as interoperability or document context | Stable interchange names and document-owned text | — |
| "Opens PSD files" | Only as a statement | Factual compatibility, in help and import messages | — |
| "Figma-class" | No | Comparison in product copy | Describe our feature directly |

Names apps have left for this RFC (for example Warp Stabilizer, Roto Brush and AudioSuite) are decided by each app under the rule above.

## Applying it

- Change our user-facing wording, not command ids, preference keys or stored values. Map stable keys to the new label.
- Keep the previous label as a search alias that isn't displayed; results show the new label.
- Translate the new label; keep another company's name in a translation only where English would keep it.

## Out of scope

The apps' own names, our interface and icon design, and patents.

## House rules

This RFC doesn't change `standards/house-rules-core.md`: the block copied into each `AGENTS.md` is at its 3 KB limit. If the rule should reach agents through that block, a follow-up can propose a line once there is room.
