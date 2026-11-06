# RFC-0011: Language data comes from craft-i18n

Spelling and hyphenation work for English and little else, and each app
does it differently. WordCraft and DesignCraft embed Moby, PhotoCraft
embeds SCOWL, DesignCraft and craft-text enable a few `hypher` languages,
DeckCraft and GridCraft read only system dictionaries, and the other apps
have none. Two apps check the same word differently. About 25 open issues
ask for a missing language, and about 10 more report that a hand-installed
dictionary isn't found.

## Decision

- craft-i18n holds the suite's **Language data**: spelling dictionaries
  (Hunspell `.aff`/`.dic`), hyphenation patterns (TeX) and the suite word
  list. Upstream files are unchanged, pinned by commit and sha256, and
  checked by CI. The code that reads them stays in craft-libs:
  craft-proof for spelling, craft-text for hyphenation
  (craft-i18n ADR 0002, 0005).
- English (en-US, en-GB) is embedded in every app from `CRAFT_I18N_DIR`,
  the same input as the catalogs. A build without it still builds.
- Other languages are downloaded only after the user opts in, through
  the font store and its single consent switch (craft-libs#223). Until
  that ships, apps fall back to system Hunspell dictionaries. A
  **Language build** (`CRAFT_LANGUAGES=de,fr`) embeds chosen languages
  for packagers and managed installs.
- What an app gets is fixed by its Pin, never `main`.
- Third-party data is allowed under craft-i18n ADR 0004's licence table.
  GPL-only data is download-only. Nothing copyleft ships until a
  maintainer signs the table off.
- Apps stop carrying their own word lists, patterns and `hypher` features.
  Each app drops its copy in the same PR that adopts craft-i18n's data,
  and lists the spelling changes as parity exceptions.

## What this asks of each app

- Read spelling through craft-proof and hyphenation through craft-text.
- Add no new dictionary, word list or pattern file of its own. Fixes go
  upstream, and the app moves its pin.
- Keep User words as they are today. They are app state, not Language data.

## Not in this RFC

Thesaurus, grammar, autocorrect, the shared User word store, the download
UI (craft-libs#223) and ICU4X segmentation data.

## Open

- Licence table sign-off (craft-i18n ADR 0004).
- craft-libs#223 decisions 1 (network) and 4 (index hosting), which
  block downloads.
- craft-text (aiongg) and craft-proof (brzuchal) owners agreeing to read
  Language data from craft-i18n.
- Should any of "Not in this RFC" be included? The thesaurus is the
  likeliest candidate. Including it means a format (e.g. MyThes
  `.dat`/`.idx`), its licences (many are LGPL or GPL), and a reader in
  craft-libs, which doesn't exist yet. It also means updating craft-i18n's
  spec, glossary and ADR 0005, which exclude it today. Each item can
  instead come back as its own RFC.

## References

- craft-i18n `plans/language-data-spec.md`,
  `plans/2026-10-23-plan-language-data.md`
- craft-i18n `docs/adr/0002`, `0004`, `0005`; `GLOSSARY.md`
- craft-i18n#8, craft-libs#18, #19, #31, #223

The craft-i18n spec, plan, ADRs 0004 and 0005 and the glossary changes
are local to benandez for now and not yet pushed; links follow once they
are.
