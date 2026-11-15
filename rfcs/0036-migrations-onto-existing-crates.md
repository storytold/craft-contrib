# RFC-0036: App copies of code that craft-libs crates already hold

The Craft apps keep their own copies of runtime code that a craft-libs crate already holds: quit
handling, file dialogs, atomic writes, font discovery, menus, ICC code, settings stores, the
`craft-fonts-build` readers, and app copies of the `craft-xlsx`, `craft-raw`, `craft-metadata`,
`craft-video` DNx and `craft-xfl` code. This RFC proposes that each copy in the table below moves onto
its crate, after the crate takes the copy's missing test cases. No new crate is needed. These moves
change shipped code, and the copies differ from their crates in permissions, retries, folder lists and
dialog behavior.

RFC-0035 (storytold/craft-contrib#55) covers the shared test and development tooling. This RFC covers
the runtime copies.

Related: storytold/craft-contrib#55, storytold/craft-contrib#15, storytold/craft-libs#2,
storytold/craft-libs#558, storytold/craft-libs#576, storytold/craft-libs#515, storytold/craft-libs#376,
storytold/craft-libs#377, storytold/craft-libs#381, storytold/craft-libs#382, storytold/craft-libs#383,
storytold/craft-libs#385, storytold/craft-libs#386, storytold/craft-libs#501, storytold/lightcraft#4244.

## Summary

Each app copy in "Copies and their crates" moves onto the craft-libs crate that already holds the same
code: `craft-quit`, `craft-dialogs`, `craft-atomic`, `craft-font-catalog`, `craft-menu`, `craft-cms`,
`craft-settings`, `craft-fonts-build`, `craft-xlsx`, `craft-raw`, `craft-metadata`, `craft-video`'s DNx
codec and `craft-xfl`. Before a copy is deleted, every test case the crate lacks is ported to the
crate. A change of behavior that a switch brings is its own pull request.

The `craft-i18n` row is listed with its copies, and this RFC does not decide it. RFC-0011
(storytold/craft-contrib#15) moves language data (spelling dictionaries, hyphenation and word lists)
into craft-i18n. It does not cover the apps' UI catalogs or their `tl!` macros, so moving the UI
catalogs needs its own decision.

## Copies and their crates, non-normative

This section is a record as of 2026-11-14. It is not updated after this RFC is accepted, and a renamed
file, a changed count or a consumer that drops out does not change what the RFC decides.

The listings were read on 2026-11-14 at vectorcraft `460d211995`, photocraft `0d3b1e8523`, designcraft
`8b169f8ee7`, lightcraft `487833ad19`, filmcraft `e4ab3c6a99`, effectcraft `f029bbadb2`, pdfcraft
`346d803080`, gridcraft `c2aba7a3f4`, wordcraft `f8c66d0692`, deckcraft `b0c77ebd6a`, soundcraft
`2688ccac88`, cadcraft `8e69890223`, figcraft `bad07e4433`, anicraft `6784cf72ce`, materialcraft
`bb7434478c`, texturecraft `386630ee85`, craft-libs `7ad4765ee5`, craft-launcher `ba56c4587f` and
craft-anything `58748a7463`. Counts made with `git grep` are lower bounds.

| Crate | Repositories that use it | Copies it replaces | Source of missing cases (tests) | First consumers |
|---|---|---|---|---|
| `craft-quit` | EffectCraft, PdfCraft, MaterialCraft, TextureCraft, craft-launcher | Their own quit handling in `on_exit`: VectorCraft `apps/vectorcraft/src/app.rs:100`; PhotoCraft `crates/ui-egui/src/app.rs:250`; DesignCraft `apps/designcraft/src/main.rs:107`; LightCraft `apps/lightcraft/src/main.rs:185`; FilmCraft `crates/ui-egui/src/app.rs:507`; GridCraft `apps/gridcraft/src/main.rs:64`; WordCraft `apps/wordcraft/src/main.rs:131`; DeckCraft `apps/deckcraft/src/main.rs:52`; SoundCraft `apps/soundcraft/src/main.rs:103`; CADCraft `apps/cadcraft/src/main.rs:127`; FigCraft `apps/figcraft/src/main.rs:132`; AniCraft `src/app.rs:3656`; Craft Anything `crates/ui-egui/src/lib.rs:2233` | The crate's own tests | DeckCraft and CADCraft |
| `craft-dialogs` | DesignCraft, MaterialCraft, TextureCraft | Direct `rfd` dependencies: VectorCraft `apps/vectorcraft/Cargo.toml:31`; PhotoCraft `apps/photocraft/Cargo.toml:47`; LightCraft `apps/lightcraft/Cargo.toml:44`; FilmCraft `apps/filmcraft/Cargo.toml:35`; EffectCraft `apps/effectcraft/Cargo.toml:24`; PdfCraft `crates/ui-egui/Cargo.toml:47`; GridCraft `apps/gridcraft/Cargo.toml:40`; WordCraft `apps/wordcraft/Cargo.toml:90`; DeckCraft `apps/deckcraft/Cargo.toml:31`; SoundCraft `apps/soundcraft/Cargo.toml:38`; CADCraft `apps/cadcraft/Cargo.toml:45`; FigCraft `apps/figcraft/Cargo.toml:31`; AniCraft `Cargo.toml:54`; craft-launcher `apps/artcraft-launcher/Cargo.toml:65`; Craft Anything `apps/craft-anything/Cargo.toml:23` | EffectCraft `apps/effectcraft/src/file_dialogs.rs` (5) and WordCraft `apps/wordcraft/src/file_dialogs.rs` (5) | EffectCraft and WordCraft |
| `craft-atomic` | DesignCraft (`Cargo.toml:99`, `crates/engine/Cargo.toml:49`; `crates/engine/src/atomic.rs:1-18` wraps `craft_atomic::atomic_write` and `atomic_write_with`) | VectorCraft `crates/format/src/atomic.rs`; PhotoCraft `crates/format/src/atomic.rs`; LightCraft `crates/catalog/src/safe_file.rs`; FilmCraft `crates/format/src/atomic.rs`; EffectCraft `crates/engine/src/settings/config.rs`; PdfCraft `crates/platform/src/staging.rs`; DeckCraft `crates/ui-egui/src/settings.rs` and `crates/engine/src/recovery.rs`; SoundCraft `crates/engine/src/io.rs`; CADCraft `crates/engine/src/atomic_write.rs`; FigCraft `crates/format/src/lib.rs`; AniCraft `crates/engine/src/recovery.rs`; MaterialCraft `crates/formats/src/file.rs`; TextureCraft `crates/formats/src/lib.rs`; craft-launcher `crates/native/src/paths.rs`; Craft Anything `crates/platform/src/fs.rs`; and further private copies in the same repositories | PhotoCraft `crates/format/src/atomic.rs` (39) | VectorCraft and FilmCraft |
| `craft-font-catalog` | WordCraft (`Cargo.toml:33`, `crates/engine/Cargo.toml:46`, used at `crates/engine/src/packs/store.rs:7`); FigCraft (`Cargo.toml:50`, `crates/engine/Cargo.toml:40`, `crates/text/Cargo.toml:31`; `crates/text/src/fontdb.rs:1330` takes the Linux and BSD font folders from `craft_font_catalog::font_dirs`) | Installed-font discovery: VectorCraft `crates/text/src/fontdb/dirs.rs`; PhotoCraft and FilmCraft `crates/text/src/fonts.rs`; DesignCraft `crates/fonts/src/fontdb/system.rs`; EffectCraft `crates/text/src/font_dirs.rs`; PdfCraft `crates/render/src/system_font.rs`; GridCraft `crates/ui-egui/src/fonts.rs`; WordCraft `crates/fonts/src/fontdb/names.rs:210` (`system_font_dirs`); DeckCraft `crates/fonts/src/fontdb.rs`; CADCraft `crates/fonts/src/ttf.rs`; FigCraft's macOS and Windows folders in `crates/text/src/fontdb.rs:1300-1325`; TextureCraft `crates/formats/src/fonts.rs` | FigCraft `crates/text/src/fontdb_tests.rs` (30), for the macOS and Windows cases | DeckCraft and WordCraft (`crates/fonts`) |
| `craft-menu` | PdfCraft, FigCraft, MaterialCraft, TextureCraft | Direct `muda` dependencies: VectorCraft `apps/vectorcraft/Cargo.toml:71`; PhotoCraft `apps/photocraft/Cargo.toml:71`; DesignCraft `apps/designcraft/Cargo.toml:48`; LightCraft `apps/lightcraft/Cargo.toml:50`; FilmCraft `apps/filmcraft/Cargo.toml:51`; EffectCraft `apps/effectcraft/Cargo.toml:51`; GridCraft `apps/gridcraft/Cargo.toml:54`; SoundCraft `apps/soundcraft/Cargo.toml:45`; CADCraft `apps/cadcraft/Cargo.toml:52`; Craft Anything `crates/ui-egui/Cargo.toml:42` | The crate's own tests | CADCraft and GridCraft |
| `craft-i18n` (not decided here) | Every app through `craft-ui`; PdfCraft, PhotoCraft, WordCraft and TextureCraft also from their UI crate | TSV catalogs with an app `tl!` macro: VectorCraft `crates/ui-egui/src/lib.rs:19`, `crates/ui-egui/src/i18n.rs` and 28 catalogs in `crates/ui-egui/src/i18n/`; CADCraft `crates/ui-egui/src/i18n.rs:433` and 18 catalogs; FilmCraft `crates/ui-egui/src/lib.rs:20` and 16 catalogs; craft-launcher `crates/ui-egui/src/lib.rs:13` and 12 catalogs in `crates/ui-egui/src/i18n/`. Rust tables: `crates/ui-egui/src/i18n.rs` in DesignCraft, GridCraft, EffectCraft, LightCraft, FigCraft and MaterialCraft | VectorCraft `crates/ui-egui/src/i18n/tests.rs` (122) | Moving the UI catalogs needs its own decision; RFC-0011 covers language data only |
| `craft-cms` | PhotoCraft, DesignCraft, PdfCraft | ICC code: VectorCraft `crates/color/src/cms.rs` and `crates/color/src/cms/` (`icc.rs`, `display.rs`, `generic.rs`, `lab.rs`); EffectCraft `crates/color/src/icc.rs`; FigCraft `crates/color/src/icc.rs` (a profile writer); TextureCraft `crates/formats/src/icc.rs`; LightCraft `crates/codecs/src/icc.rs` | VectorCraft `crates/color/src/cms/tests.rs` and `tests_icc.rs` (40) | EffectCraft (7) and LightCraft (6) |
| `craft-settings` | FigCraft (`Cargo.toml:57`, `apps/figcraft/Cargo.toml:24`) | Settings stores: VectorCraft `apps/vectorcraft/src/prefs.rs`; LightCraft `apps/lightcraft/src/prefs.rs` (`ui.json`, with damaged files set aside at `:79`); EffectCraft `crates/engine/src/settings/config.rs`; DeckCraft `crates/ui-egui/src/settings.rs`; AniCraft `src/prefs.rs` | VectorCraft `apps/vectorcraft/src/prefs.rs` (16) | DeckCraft and LightCraft |
| `craft-fonts-build` | Every app through `craft-ui` only | `build.rs` readers of `CRAFT_FONTS_DIR`: VectorCraft `crates/text/build.rs:79`; PhotoCraft `crates/text/build.rs:66`; DesignCraft `crates/fonts/build.rs:30`; LightCraft `crates/engine/build.rs:64`; FilmCraft `crates/text/build.rs:56`; PdfCraft `crates/fonts/build.rs:41`; GridCraft `crates/ui-egui/build.rs:78`; WordCraft `crates/fonts/build.rs:47`; DeckCraft `crates/fonts/build.rs:83`; CADCraft `crates/fonts/build.rs:41`; FigCraft `crates/text/build.rs:131`; TextureCraft `crates/ui-egui/build.rs:49`; Craft Anything `crates/ui-egui/build.rs:83` (each `fn craft_fonts`); AniCraft `build.rs:38` (`discover`) | GridCraft `crates/ui-egui/src/craft_fonts_policy.rs` (8) | DesignCraft and CADCraft |
| `craft-xlsx` | DesignCraft | GridCraft `crates/xlsx` (`gridcraft-xlsx`, `Cargo.toml:22`), which shares nine file names with craft-libs `crates/xlsx/src` (`lib.rs`, `drawing.rs`, `package.rs`, `pivot.rs`, `richdata.rs`, `styles.rs`, `testing.rs`, `write.rs`, `xlsb.rs`) and depends on `craft-xls` (`Cargo.toml:43`) but not on `craft-xlsx` | `gridcraft-xlsx`'s `crates/xlsx/src/tests.rs` and `crates/xlsx/src/tests/` | GridCraft and `craft-xlsx` |
| `craft-raw` | PhotoCraft, LightCraft | LightCraft `crates/raw` (`lightcraft-raw`, `Cargo.toml:45`), which depends on `craft-raw` (`crates/raw/Cargo.toml:13`) since storytold/lightcraft#4244 removed its `crates/raw/src/vendor/` decoders, and keeps 7 files with the same names as files in `craft-raw`'s `crates/raw/src` (5 modules besides `lib.rs` and `tests.rs`: `dngwrite.rs`, `gaintable.rs`, `opcodes.rs`, `profile.rs` and `semantic.rs`) | LightCraft `crates/raw/tests/` (21 files) | LightCraft and `craft-raw` |
| `craft-metadata` | LightCraft | LightCraft `crates/meta` (`lightcraft-meta`, `Cargo.toml:47`), which depends on `craft-metadata` (`crates/meta/Cargo.toml:13`) since storytold/lightcraft#4244 and keeps `exif.rs`, `focus.rs`, `iptc.rs` and `xmp.rs`, the same file names as in craft-libs `crates/metadata/src` | LightCraft `crates/meta/tests/` and the modules' own tests | LightCraft and `craft-metadata` |
| `craft-video` (`dnx` feature) | FilmCraft, EffectCraft, DeckCraft, SoundCraft | FilmCraft `crates/dnx` (`filmcraft-dnx`, `Cargo.toml:22`), whose eight modules have the same file names as craft-libs `crates/video/src/dnx/` (`bits.rs`, `dct.rs`, `decode.rs`, `encode.rs`, `ffmpeg_probe.rs`, `header.rs`, `spec_tables.rs`, `tables.rs`) | FilmCraft `crates/dnx/tests/it` (7 files) | FilmCraft and `craft-video` |
| `craft-xfl` | AniCraft (`Cargo.toml:35`, `crates/xfl/Cargo.toml:16`) | AniCraft `crates/xfl/src/edges.rs`, the edge-string parser that craft-libs `crates/xfl/src/edges.rs` also holds | AniCraft `crates/xfl/src/tests.rs` and craft-libs `crates/xfl/tests/support/fla.rs` | AniCraft and `craft-xfl` |

- **craft-quit.** VectorCraft, MaterialCraft and EffectCraft each have an unsaved-changes prompt. A
  switch replaces the app's own quit handling and keeps `on_exit` for what the app saves at exit, as
  `craft-quit`'s documentation asks (`crates/quit/src/lib.rs:42-44`). DeckCraft, for one, calls
  `save_prefs` there.
- **craft-dialogs.** EffectCraft's guarded `apps/effectcraft/src/file_dialogs.rs` is the closest to it.
- **craft-atomic, then craft-settings.** The copies of a temporary-file, flush and rename write differ in
  Windows retries, symlink handling, permissions and folder sync. DesignCraft already writes through
  `craft-atomic`, and `craft-settings` is used by FigCraft only. craft-launcher
  `crates/native/src/paths.rs:132`, CADCraft `crates/engine/src/atomic_write.rs:282` and `:320`, and
  CADCraft `apps/cadcraft-cli/src/lib.rs:171` are the tests `craft-atomic` passes before a repository
  switches. `craft-settings` stores `config.toml` and reads an older file only when the app lists it as
  a `craft_settings::LegacyFile`, and it imports only the first listed file that parses. A switch to
  `craft-settings` reads every settings file the old store wrote, in the same pull request, and combines
  them where there are several (DeckCraft keeps `ui.json` and `prefs.json`; LightCraft keeps `ui.json`).
  A test checks that each value written by the old store survives the switch.
- **craft-font-catalog and craft-ui's `UiFonts`.** WordCraft uses `craft-font-catalog` in
  `crates/engine` and keeps its own folder list in `crates/fonts`. FigCraft takes its Linux and BSD
  folders from the crate and keeps its macOS and Windows folders. No app uses
  `craft_ui::fonts::UiFonts`; DeckCraft, MaterialCraft, TextureCraft and VectorCraft define their own
  types of that name. VectorCraft's `crates/text/src/fontconfig.rs` and `suitcase.rs` have the same
  module names as `craft-font-catalog`'s.
- **craft-i18n.** VectorCraft, CADCraft, FilmCraft and craft-launcher keep TSV catalogs with their own
  `tl!` macro, and DesignCraft, GridCraft, EffectCraft, LightCraft, FigCraft and MaterialCraft keep Rust
  tables. RFC-0011 (storytold/craft-contrib#15) moves language data (spelling dictionaries, hyphenation
  and word lists) to craft-i18n and does not cover UI catalogs or `tl!`. Moving the UI catalogs needs
  its own decision, in its own RFC.
- **craft-cms.** `craft-cms`, VectorCraft and FigCraft use moxcms 0.8. LightCraft's `Cargo.toml` names
  moxcms 0.9 (`:94`), and its lock holds only 0.8.1.
- **craft-fonts-build.** The fourteen `build.rs` readers in the table switch to it with the embedded font
  list unchanged.
- **Format crates that an app also keeps.** The `craft-xlsx`, `craft-raw`, `craft-metadata`,
  `craft-video` and `craft-xfl` rows follow the porting rule in "How the code moves". WordCraft's
  `wordcraft-docx` (`crates/docx/src/lib.rs` delegates to `craft-docx`) and PdfCraft's `pdfcraft-cos`
  (`crates/cos/src/lib.rs` is `pub use craft_pdf::cos::*;`) already use their craft-libs crates, so
  only their test suites move.

## How the code moves

Each move is made in three kinds of pull request:

1. A craft-libs pull request adds to the crate every case from the source of missing cases that the
   crate's tests lack, with the case's original test name. Where the copy behaves differently from the
   crate (a permission, a retry, a folder list), this pull request also adds that behavior to the crate
   as an option, so the app can switch without changing what it does.
2. One pull request per repository switches it to the crate, with the options that keep its current
   behavior, and deletes its copy. The repository's tests pass and skip as before, and the pull request
   lists both counts. When the crate stores data in a different place or format than the copy did, the
   same pull request reads the data the copy wrote.
3. A later change of the app's behavior, such as dropping an option to take the crate's default, is its
   own pull request with a test that fails without it. It lists the tests whose result changes.

Where a repository keeps its own copy of a craft-libs crate's code, its test suite has diverged from
the crate's. Before such a copy is deleted, the repository's tests and `tests/support` are compared
with the crate's, and every case the crate lacks is ported to the crate first.

A switch onto a crate merges with the open pull requests on that crate or waits for them.

## Open pull requests on these crates, non-normative

This list is a record as of 2026-11-14.

- `craft-atomic` and `craft-settings`: storytold/craft-libs#558 (dylanmtaylor) changes
  `crates/atomic/src/lib.rs` and `crates/settings/src/recovery.rs` to give a replaced file's group bits
  only to its group, which bears on the permissions in which the copies differ.
  storytold/craft-libs#576 (CypherNaught-0x, draft) adds transactional updates to `craft-settings`, and
  #515 (SDG-Den, draft) adds a settings overlay and a shared palette.
- `craft-font-catalog`: aiongg's drafts storytold/craft-libs#376, #381, #382, #383, #385 and #386 rework
  `crates/font-catalog` (document font scopes, an app module, a face cache, a font store and manager),
  and #377 (aiongg, draft) adds a `craft-ui` system font fallback over the catalog. All seven conflict
  with `main`.
- `craft-menu`: storytold/craft-libs#501 (ahmedhosnypro, draft) mirrors the menu bar for right-to-left
  languages.

## What this RFC decides

- Each app copy in "Copies and their crates" moves onto its craft-libs crate, except the `craft-i18n`
  row, which needs its own decision.
- The three kinds of pull request in "How the code moves", with the missing cases ported to the crate
  before a copy is deleted.

The affected repositories are craft-libs and the Craft applications with a copy in the table:
VectorCraft, PhotoCraft, DesignCraft, LightCraft, FilmCraft, EffectCraft, PdfCraft, GridCraft, WordCraft,
DeckCraft, SoundCraft, CADCraft, FigCraft, AniCraft, MaterialCraft, TextureCraft, craft-launcher and
Craft Anything.

Per RFC-0000 this is a 3-day comment period change.
