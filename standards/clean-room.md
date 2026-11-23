# Clean room, assets and fixtures

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as in RFC 2119 and RFC 8174.

These rules apply in every Craft repository, with the same restrictions everywhere. `AGENTS.md` carries the short form and links here. Changing a rule needs an RFC.

## Where knowledge comes from

1. You MUST learn only from:
   - public specifications and conformance material;
   - public documentation: help pages, published screenshots, tutorial videos;
   - papers and textbooks;
   - behaviour specifications written as "Observing a proprietary app" describes.

   You SHOULD cite the source in the module or crate docs. For a format or codec, you SHOULD record the specification and its edition, and the pull request SHOULD cite the sections it implements. A statement that code is clean-room is not evidence; a reviewer who finds code shaped like another implementation SHOULD ask about it before merging.
2. You MUST learn behaviour and ideas only. You MUST NOT copy another product's code, shaders, presets, string tables, help text, UI wording, icons, artwork or pixels, in tests and fixtures too. Feature names follow [naming.md](naming.md).
3. You MUST NOT use remembered knowledge of a proprietary product's internals (SDK class names, IDs, persistence details). A fact that no public specification, behaviour specification or corpus evidence shows MUST NOT go in.
4. You MUST NOT read, copy or link GPL, LGPL or AGPL code. Copyleft tools such as ffmpeg MAY run as separate processes that generate fixtures or check output in tests. You MUST NOT link, bundle or ship them.
5. You SHOULD implement a format or codec from its specification and conformance material. You SHOULD NOT implement it from another implementation's source outside storytold, whatever its licence. Permissively licensed crates MAY be dependencies under [licences.md](licences.md); shared code comes from craft-libs.
6. You SHOULD write translations from the meaning of the English text and SHOULD NOT consult another product's localisation. Language data (dictionaries, hyphenation patterns) SHOULD come unchanged from open-licensed upstream files through craft-i18n.
7. You MAY follow another product's layout, arrangement and conventions. Every icon, control, colour and font MUST be drawn or chosen by us; you MUST NOT trace or sample anything from its screens.

## Observing a proprietary app

Two roles keep what is learned from the app apart from the code. One agent session MUST NOT take both.

1. **Observer.** An agent that operates the app. It MUST only write a behaviour specification. It:
   - MUST run the proprietary app only with inputs we generate or a contributor owns;
   - SHOULD write a behaviour specification of what a user sees and does: screens, controls, results;
   - SHOULD screenshot only the app's window, by its window id, and SHOULD NOT capture the rest of the desktop;
   - SHOULD NOT capture the user's own documents, account, store, cloud, home or recent-files screens;
   - MUST NOT instrument the app: no tracing, debugging, memory or traffic inspection;
   - MUST NOT read, disassemble, decompile or copy anything inside the app's install or bundle, and MUST NOT copy its fonts, presets, templates, profiles, plug-ins or sample files.
2. **Implementer.** An agent that writes code. It MUST NOT operate the app or read its install. It works from behaviour specifications and their screenshots, public documentation and published format specifications.
3. **Measurements and fitted models.** You MAY fit parameters of our own documented mathematical model (such as a 3×3 colour matrix or a global tone curve) to pairs of inputs and exported outputs from a proprietary app, provided:
   - the inputs are files a contributor owns or we generate;
   - the app is operated as a black box by an Observer agent with settings recorded in a behaviour specification;
   - no code, binary, shader, profile, LUT or asset inside the app's install is inspected or copied;
   - an Implementer agent who has not operated the app fits our own model to the decoupled data;
   - the fitted numbers and their provenance are recorded in `ATTRIBUTION.md`;
   - the exported images and intermediate captures remain in the gitignored `plan/` folder and are never committed or published.
4. **Captures.** Screenshots, recordings and observation notes stay in the gitignored `plan/` folder. You SHOULD NOT commit, publish, link, embed or trace them, and SHOULD NOT attach them to issues or pull requests.

## Assets

1. An asset is any non-code material the project includes, ships, embeds, generates for publication or shows in its documentation: images, icons, cursors, logos, screenshots, fonts, colour profiles and LUTs, presets, templates, themes, sounds, samples, music, video, 3D models, sample documents. It includes bytes embedded in code (`include_bytes!`, base64, data URIs) and data hard-coded to reproduce someone else's work (traced point lists).
2. Every asset MUST be one of:
   - original work by a contributor who owns it and licenses it under the repository's licence;
   - under a licence on [licences.md](licences.md), taken from its original source (the author's repository or release, not a mirror), with the exact version recorded.
3. You MUST NOT use an asset from a proprietary product, its installer, website or documents: icons, UI artwork, fonts it installs, presets, templates, stock media, sample projects. This includes traced, redrawn, recoloured or lookalike copies, also as a placeholder or a reference file.
4. Also excluded: system fonts that are not openly licensed, vendor emoji artwork, stock photos, "free for personal use" material, and anything whose licence you cannot show from its original source.
5. Fonts follow [licences.md](licences.md).
6. Art drawn or generated in code, synthesized sound and generated demo content are original work; you SHOULD record them as such.
7. The ArtCraft name and logos in `docs/brand/` are trademarks, not open source, under `docs/brand/LICENSE-brand.txt` (`LicenseRef-ArtCraft-Trademark`; pdfcraft's records still say `LicenseRef-Storyteller-Trademark` until its move to the shared `AGENTS.md`). You SHOULD use them unmodified and only for the project; forks SHOULD remove them. They are the only asset not under an open licence, and they allow no other company's marks.

## Fixtures and corpus files

A fixture is a file tests read; a corpus file is a fixture kept in a corpus repository. A fixture MUST be one of:

1. A file we generate with a committed script from our own inputs.
2. A file others made for their own purposes and published (templates, test suites, archives), obtained from its original source. It MAY be committed only when its licence is on [licences.md](licences.md); otherwise a committed manifest lists it (pinned URL, sha256, licence) for tests to fetch.
3. A file a person made, with any app they hold a licence for, when that person owns it, releases it under a licence on [licences.md](licences.md), and it embeds no third-party assets (fonts, stock media, presets, templates) and no personal data. Such a person MAY make the files by running a committed script that drives the app; an agent MUST NOT run that script.

You SHOULD NOT ask anyone to make files for the project. You MUST NOT commit a file from anyone's disk that is not one of the three. Every fixture's provenance MUST be recorded in the repository's attribution or manifest file: source (URL and version, or script and inputs, or maker), licence and how it was obtained.

## Attribution

1. Every asset MUST be recorded in the repository's attribution file (`ATTRIBUTION.md`, or the file `## This repository` names) in the same change that adds it: path, title, author, source with version, licence, and how it was made or changed. Assets bundled through a dependency or fetched at build time get a row too.
2. You SHOULD keep the licence text next to the asset when the licence requires it, and the required notices in `NOTICE`.
3. Where the repository has an asset check (`cargo xtask assets`), it SHOULD pass. You SHOULD NOT weaken it to pass; fix the record or remove the asset.

## Screenshots, demo and test media

1. Screenshots in a repository SHOULD show our own app, or another open project, with content these rules allow.
2. Demo and test media SHOULD be generated in code, or be public domain or under a licence on [licences.md](licences.md) with their source recorded. They SHOULD NOT contain photos of people, anything from anyone's disk or screen, or personal details in contents, metadata, layer or file names.

## When something is wrong

1. If an asset's or fixture's origin or licence is unclear, you SHOULD leave it out and draw, generate or compose a replacement, or ask on the pull request.
2. If you find an asset or fixture that breaks these rules, you SHOULD remove it from the repository and every output (docs, screenshots, generated files) in a pull request, and say on that pull request what it was and where it came from.
