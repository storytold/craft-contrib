# RFC-0011: Language data comes from craft-i18n

Updated by: [RFC-NNNN](NNNN-hyphenation-through-craft-proof.md)

Spelling and hyphenation work for English and little else, and each app
does it differently. WordCraft and DesignCraft embed Moby, PhotoCraft
embeds SCOWL, DesignCraft and craft-text enable a few `hypher` languages,
DeckCraft and GridCraft read only system dictionaries, and the other apps
have none. Two apps check the same word differently. About 25 open issues
ask for a missing language, and about 10 more report that a hand-installed
dictionary isn't found.

The work is already under way, so this RFC records it as the suite's
decision. As of 2026-11-15 craft-i18n holds the data, craft-proof reads
it, the first Language packs are published, and WordCraft has cut over
(see "Where it stands").

## Decision

- craft-i18n holds the suite's **Language data**: spelling dictionaries
  (Hunspell `.aff`/`.dic`), hyphenation patterns (TeX) and the suite word
  list. Upstream files are unchanged, pinned by commit and sha256, and
  checked by CI. The code that reads them stays in craft-libs:
  craft-proof for spelling, craft-text for hyphenation
  (craft-i18n ADR 0002, 0005).
- English (en-US, en-GB, en-CA, en-AU from SCOWL, plus the Moby
  hyphenation list and our own supplements) is committed and embedded
  from `CRAFT_I18N_DIR`, the same input as the catalogs. A build without
  it still builds, with no embedded Language data. `CRAFT_I18N_REQUIRED=1`,
  which releases set, makes a missing checkout an error.
- Other languages are **Language packs**, one per locale. craft-i18n's
  manifest pins each upstream file; CI fetches, checks and publishes the
  packs as `language-packs-<n>` release assets with an `index.json`. An
  app pins that index by URL and sha256 and downloads a pack through the
  font store, only after the user opts in: their own click turns on the
  font store's single download setting. Agents can't turn it on.
- A build chooses its embedded languages at compile time. A **Language
  build** names them in `CRAFT_PROOF_EMBED` (`de,pl`, `all`, or `en` for
  English only), read by craft-proof's `build.rs`. This is how a web
  build or the headless engine, which can't download, ships exactly the
  set it names. The choice is an environment variable, not one Cargo
  feature per language: Cargo can't reach data in craft-i18n, which apps
  never list in `Cargo.toml` (ADR 0002), and Cargo features can't add a
  language after install (ADR 0005). Downloads are an addition for builds
  that want them.
- Offline, or without the language: English always works. An installed
  pack keeps working with no network. A spelling dictionary is looked up
  in the user's chosen dictionaries folder, then an installed pack, then the
  system's Hunspell dictionaries. With none of those, that language isn't
  spell-checked or hyphenated, and the app offers its pack, when one
  exists, with its size. Nothing is requested until the user opts in.
- Size: the English files embedded at craft-i18n `866aff71` are
  2,629,371 bytes uncompressed. The carry-over (below) adds 326,723
  bytes. What that adds to each app's binary, compressed and on web, is
  to be recorded by each app when it cuts over.
- What an app gets is fixed by its Pin, never `main`.
- Third-party data is allowed under craft-i18n ADR 0004's licence table,
  signed off on 2026-11-01. Only Embeddable files are committed or
  embedded. Download-only (LGPL, GPL, CC-BY-SA) data ships only in packs.
  Excluded data is recorded and never published.
- Apps stop carrying their own word lists, patterns and `hypher` features.
  Each move is two PRs, in order. The craft-i18n PR adds the data and
  lands first. The app PR then bumps its pin, drops its own copy and
  lists the spelling changes as parity exceptions. Each PR links the
  other.

## What this asks of each app

- Read spelling through craft-proof and hyphenation through craft-text.
- Set `CRAFT_I18N_DIR` at a pin that has `language-data/`, and
  `CRAFT_I18N_REQUIRED=1` in releases.
- Add no new dictionary, word list or pattern file of its own. Fixes go
  to craft-i18n or upstream, and the app moves its pin.
- Keep User words as they are today. They are app state, not Language data.

## Where it stands

- craft-i18n `language-data/`: English, and WordCraft's carry-over (de,
  pl, pt, it hyphenation, Russian extra words). The manifest lists 80
  locales: 75 published, 2 hidden until craft-proof can use them (mn,
  th), 3 Excluded (eu, ug, uk). `language-packs-2` is the latest release.
- craft-proof (craft-libs#482, #483, #485, #495) embeds and reads it; the
  font store has a `language-pack` kind (craft-libs#486).
- WordCraft cut over (wordcraft#5202) and downloads packs from
  `language-packs-2` (wordcraft#5208). VectorCraft uses craft-proof.
  DesignCraft (#2286, #2327) and GridCraft (#723) are in review.

## Not in this RFC

Thesaurus, grammar, autocorrect, the shared User word store, the Fonts
tab and the rest of the font store (craft-libs#223) and ICU4X
segmentation data.

## Open

- The carry-over leaves the default build now that packs ship
  (`CRAFT_PROOF_EMBED`'s default becomes English only).
- A Language build can name only committed data today. Embedding a
  published pack at compile time is to be added.
- craft-text still bundles its own English patterns and `hypher`
  features. Reading craft-i18n's needs a craft-libs PR.
- The suite word list (`language-data/words.txt`) doesn't exist yet.
- craft-libs#223 is still open. Language packs follow its proposals for
  decisions 1 (network) and 4 (index hosting: craft-i18n release
  assets); the issue should record that.
- Whether craft-common-actions' release workflow sets `CRAFT_I18N_DIR`
  and `CRAFT_I18N_REQUIRED=1` (craft-i18n `docs/integration.md`).
- Should any of "Not in this RFC" be included? The thesaurus is the
  likeliest candidate. Including it means a format (e.g. MyThes
  `.dat`/`.idx`), its licences (many are LGPL or GPL), and a reader in
  craft-libs, which doesn't exist yet. It also means updating craft-i18n's
  glossary and ADR 0005, which exclude it today. Each item can instead
  come back as its own RFC.

## References

- craft-i18n [ADR 0002](https://github.com/storytold/craft-i18n/blob/main/docs/adr/0002-runtime-in-craft-libs-data-in-craft-i18n.md),
  [ADR 0004](https://github.com/storytold/craft-i18n/blob/main/docs/adr/0004-third-party-language-data.md),
  [ADR 0005](https://github.com/storytold/craft-i18n/blob/main/docs/adr/0005-language-data-lives-here.md),
  [Language data format](https://github.com/storytold/craft-i18n/blob/main/docs/language-data-format.md),
  [Integration](https://github.com/storytold/craft-i18n/blob/main/docs/integration.md),
  [`language-data/README.md`](https://github.com/storytold/craft-i18n/blob/main/language-data/README.md),
  [Glossary](https://github.com/storytold/craft-i18n/blob/main/GLOSSARY.md)
- craft-i18n#8, #103, #104, #106, #107; craft-libs#18, #19, #31, #223,
  #482, #485, #486; wordcraft#5202, #5208
