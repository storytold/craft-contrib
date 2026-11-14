# RFC-0035: Shared crates for tests and development tools

Every Craft repository carries its own copies of the same test and build helpers: temporary folders
removed on drop, skip-or-fail rules for missing programs and fonts, external test programs, GPU and
other test locks, image comparison, CLI and MCP test clients, seeded random input, egui input steps,
and the `cargo xtask` commands for the gate, layers, assets, versions and icons. Many copies started
from one source and have since diverged, and several carry a fault that another copy has fixed. This
RFC proposes one family of craft-libs crates and features for this code. Each part takes its code from
the repository that already has a tested version, names its first consumers, and replaces the copies
one repository at a time with the same test results. The family follows candidate 11, "Shared tooling
and tests", of the maintainers' source audit (storytold/craft-libs#2).

Related: storytold/craft-libs#2, storytold/craft-libs#9, storytold/craft-libs#115,
storytold/craft-libs#138, storytold/craft-libs#139, storytold/craft-libs#163, storytold/craft-libs#1373,
storytold/craft-libs#172,
storytold/craft-libs#701, storytold/craft-libs#989, storytold/craft-libs#1066, storytold/craft-libs#1112,
storytold/craft-libs#504, storytold/craft-libs#437, storytold/craft-libs#616, storytold/craft-libs#698,
storytold/craft-common-actions#124, storytold/craft-common-actions#185,
storytold/craft-common-actions#225, storytold/craft-contrib#10, storytold/craft-contrib#15,
storytold/craft-contrib#18, storytold/craft-contrib#21, storytold/craft-contrib#23,
storytold/craft-contrib#30, storytold/craft-contrib#39, storytold/craft-contrib#41,
storytold/craft-contrib#43, storytold/craft-contrib#46, storytold/craft-contrib#50,
storytold/craft-contrib#52, storytold/craft-contrib#54, storytold/lightcraft#3083, storytold/lightcraft#3096,
storytold/pdfcraft#6124, storytold/pdfcraft#6188, storytold/pdfcraft#7173,
storytold/photocraft#10189, storytold/designcraft#1980.

## Summary

The family has four parts:

1. `craft-testkit`, a new test-support crate. Its core modules use `std` and one small dependency: `temp`,
   `skip`, `oracle`, `fixtures`, `serial`, `bin`, `deadline`, `rng`, `hostile`, `fonts` and `guards`.
   Its feature modules add one dependency each: `image` (PNG), `prop` (proptest), `corpus` (SHA-256),
   `mcp` (JSON) and `log`. A `geom` feature adds `craft-geom`, `kurbo` and `proptest` for the geometry
   assertions and strategies of RFC-0030.
2. UI test support in the existing `craft-ui` crate: an `input` module and a `snapshot` module in
   `craft_ui::testing`, beside the `Screens`, `ime_events`, `compose` and `DockDriver` helpers that it
   already ships and that no app uses yet.
3. Test fixtures in the craft-libs format crates: one feature name, `test-support`, for every crate that
   exposes fixture writers, and adoption of the writers that already exist (`craft-pdf`, `craft-tiff`,
   `craft-psd`, `craft-audio`).
4. `craft-xtask`, a new library of `cargo xtask` commands: the gate's step runner, `layers`, `version`,
   `ico` and `icns`, `assets`, a display-name scanner, pinned downloads, `stats` and pinned-dependency
   checks. Each app keeps its own `cargo xtask` binary and its app-only commands.

The `guards` module is the one storytold/craft-libs#139 proposes. It comes from
storytold/craft-libs#1373, which supersedes zachshallbetter's storytold/craft-libs#163 and keeps his
commit with its authorship. The time-scaled waits and the watchdog of `deadline` come from THEtheChad's
LightCraft pull requests and are added after they merge.

The section "Candidates for later RFCs" lists runtime code that the apps also duplicate, as candidates
for later RFCs. The RFC also decides that the apps' copies of code that a craft-libs crate already
holds (`craft-quit`, `craft-dialogs`, `craft-atomic`, `craft-font-catalog`, `craft-menu`, `craft-i18n`,
`craft-cms`, `craft-settings`, `craft-fonts-build`, `craft-xlsx`, `craft-raw`, `craft-metadata`,
`craft-video`'s DNx codec and `craft-xfl`) move onto that crate after the crate takes their missing
test cases.

## What exists

The listings below were read on 2026-11-14 at vectorcraft `460d211995`, photocraft `0d3b1e8523`,
designcraft `8b169f8ee7`, lightcraft `487833ad19`, filmcraft `e4ab3c6a99`, effectcraft `f029bbadb2`,
pdfcraft `346d803080`, gridcraft `c2aba7a3f4`, wordcraft `f8c66d0692`, deckcraft `b0c77ebd6a`,
soundcraft `2688ccac88`, cadcraft `8e69890223`, figcraft `bad07e4433`, anicraft `6784cf72ce`,
materialcraft `bb7434478c`, texturecraft `386630ee85`, craft-libs `7ad4765ee5`, craft-launcher
`ba56c4587f`, craft-anything `58748a7463`, craft-common-actions `f76646c68b`, craft-i18n `e56d2624`,
craft-contrib `ce8827d4`, craft-ai-integrations `e8b31cb059`, craft-extensions `964c71574a`, craft-fonts
`c1db6d1a8e`, winprint.rs `e90c444476`, craft-pr-review-tool `197a26dc9f`, craft-repo-status-app
`b4a2f85f56` and indd-utils `136f973a9d`. Counts made with `git grep` are lower bounds. Line
references into files that changed after those commits were read again on 2026-11-14 at lightcraft
`6b2b7e6d11`, pdfcraft `eead454105`, photocraft `d6557b9a8b` and vectorcraft `107078ad10`.

Of the last seven, craft-ai-integrations holds code this RFC uses: `crates/ai-mcp/src/apps/child.rs` is
an MCP stdio client for each app's `<app>-cli mcp` (`Conn`, `spawn`, `initialize`, `request`, `notify`
and `call_tool`, a `MAX_LINE` cap at `:20`, a deadline on every wait, protocol `2025-06-18` at `:36`),
with 14 tests in `child_tests.rs`, and `crates/ai-mcp/src/mcp.rs` is an MCP stdio server. The other
six have no `xtask/` crate and no testkit crate.

### Test-support crates

- **VectorCraft.** `crates/testkit` (`vectorcraft-testkit`) is a dev-dependency of 11 packages.
  `lib.rs` holds `temp_dir(tag)`, a per-process private folder: mode 0o700 on Unix, and it fails the
  test (panics) for a symlink, a folder other users can reach or a folder owned by another user
  (`lib.rs:48-74`; the crate allows `unwrap_used`, `expect_used` and `panic` at `:23`).
  storytold/vectorcraft#4737 made it start empty in each process, and storytold/vectorcraft#5419 added
  the owner check. `lib.rs` also holds `TempDir`, a per-test folder removed on drop. `raster.rs` compares
  images (`diff`, `perceptual_diff`, and the function `assert_similar` at `:165`), reads PNG through the
  `image` crate (`:20-21`) and renders VectorCraft documents (`render_region`, `render_artboard`,
  `render_view` at `:55-65`). The other modules hold document fixtures, proptest strategies, invariants,
  geometry assertions and hand-written PDF, Illustrator, `.ase`, TIFF and font files.
- **PhotoCraft.** `crates/testkit` (`photocraft-testkit`) holds `gpu_lock()` (`gpu.rs:32`), `Sweep`
  (`sweep.rs`) and benchmark helpers (`perf.rs`).
- **LightCraft.** `crates/testkit` (`lightcraft-testkit`) holds a `skip!` macro and `record()`, which
  print `skipped (<need>): <reason>` and append one JSON line to the file `LIGHTCRAFT_SKIP_LOG` names.
  It has no required variable. Open storytold/lightcraft#3083 adds `deadline.rs` and `watchdog.rs`.
- **FilmCraft.** `crates/testkit` (`filmcraft-testkit`) holds program discovery (`oracle.rs`, with
  `find_tool_in` at `:41` taking the explicit path and `PATH` as arguments), the `require_ffmpeg!`,
  `require_ffprobe!` and `require_oracles!` macros (`lib.rs:32`, `:49`, `:66`), fixture publication
  (`fixtures.rs`), golden images with `FILMCRAFT_BLESS` and a `.attribution` sidecar written with
  `chrono` (`golden.rs:7`, `:196-241`; `Cargo.toml:15-17` names `png = "0.18"` and `chrono`), a hang
  guard (`deadline.rs:19-23`, `FILMCRAFT_HOSTILE_TIMEOUT_SECS`), `cuts.rs` and a float WAV writer
  (`wav.rs`). FilmCraft's `xtask` depends on it (`xtask/Cargo.toml:13`).
- **PdfCraft.** `crates/testkit` (`pdfcraft-testkit`) holds the `Pdf` builder. `crates/oracle`
  (`pdfcraft-oracle`) finds qpdf, pdftoppm and pdftotext (`find` at `lib.rs:127`), and its `or_skip!`
  macro expands its `assert!` in the calling test. `pixels.rs` compares gray rasters. PdfCraft's
  `xtask` depends on `pdfcraft-oracle` (`xtask/Cargo.toml:12`).
- **craft-libs.** `crates/media-testkit` (`craft-media-testkit`) finds ffmpeg and ffprobe, skips or
  fails under `CRAFT_MEDIA_REQUIRE_ORACLES`, makes private folders (`private_dir()`), publishes and
  generates fixtures, runs `with_deadline` under `CRAFT_TEST_TIMEOUT_SECS` (`lib.rs:221-222`) and holds
  the hostile-input hammer `hostile.rs`. `hostile.rs` was ported from storytold/filmcraft#1794 by
  storytold/craft-libs#989, and storytold/craft-libs#1066 ported the rest of #1794's container tests.
  craft-media, craft-cfb, craft-audio and craft-video depend on the crate.
  `crates/audio/tests/codecs/testkit.rs` wraps it and keeps its own `fixtures::generate` (`:55-69`) and a
  `generate_and_report` stub that only calls `generate()` (`:72-74`). `crates/pdf/tests/cos/oracle.rs`
  is a port of `pdfcraft-oracle`.
- **Craft Anything.** `crates/testkit` (`craft-anything-testkit`) builds reverse-engineering samples.

DeckCraft and CADCraft (`xtask/src/layers.rs:15-17`), DesignCraft (`xtask/src/layers.rs:17`, `:53`,
`:141`) and WordCraft (`xtask/src/layers.rs:17`, `:51`, `:139`) reserve a testkit layer class that
other crates may use only as a dev-dependency, and none of the four has a `crates/testkit`.
DesignCraft also declares `designcraft-testkit` at `Cargo.toml:58` (storytold/designcraft#1980).
GridCraft, FigCraft, MaterialCraft and TextureCraft have no testkit class.

### Helpers copied between repositories

| Helper | Copies | Repositories |
|---|---|---|
| Test-only drop guards that remove a folder | about 74 | LightCraft about 25, FigCraft 8, PdfCraft 7, CADCraft 5, PhotoCraft 5, craft-libs 5, WordCraft 4, FilmCraft 3, VectorCraft 2 outside its testkit, MaterialCraft 2, SoundCraft 2, EffectCraft 2, DeckCraft 1, DesignCraft 1, TextureCraft 1, craft-launcher 1 |
| `static NAME: Mutex<()>` test locks | about 110 | 16 repositories; VectorCraft 20 and PhotoCraft 18 lead |
| GPU test locks outside a testkit | 11 | PhotoCraft (`crates/gpu/tests/it/main.rs:5`), EffectCraft 4, PdfCraft 2, WordCraft, LightCraft, AniCraft, DeckCraft |
| `NEXTEST_BIN_EXE_<name>` lookups | 13 | DeckCraft, SoundCraft 2, EffectCraft 4, FilmCraft, LightCraft, PhotoCraft 2, VectorCraft 2 |
| MCP stdio clients in CLI tests | 12 repositories | CADCraft, DeckCraft, DesignCraft, EffectCraft, FigCraft, FilmCraft, GridCraft, LightCraft, PdfCraft, PhotoCraft, SoundCraft, VectorCraft |
| Seeded test PRNGs (`next`, `below`) | about 75 files | craft-libs 35, GridCraft 7, FigCraft 7, FilmCraft 5, LightCraft 5 and 8 more |
| `ProptestConfig` uses | about 90 | craft-libs 32, VectorCraft 29, LightCraft 9 and 6 more |
| egui_kittest input helpers (`fn click`, `drag`, `key`, `type_text` and others on `&mut Harness`) | about 730 in about 410 files | 17 repositories; PhotoCraft 232, EffectCraft 169, GridCraft 110 |
| UI screenshot folder variables | about 47 names | FilmCraft, MaterialCraft, TextureCraft, EffectCraft, PhotoCraft, craft-launcher, PdfCraft, craft-libs |
| `fn psnr` | about 41 | craft-libs 17, FilmCraft 12, LightCraft 6, EffectCraft 5, PhotoCraft 1 |
| Hand-written RIFF/WAVE fixtures | about 20 | FilmCraft, EffectCraft, DeckCraft, SoundCraft |
| Test files with a hand-written `%PDF-1.x` header | about 310 | PdfCraft 209, craft-libs 34, VectorCraft 33, DesignCraft 15 and 7 more |
| Thread-local warning capture (`test_log.rs`, blob `745ce21fe1`) | 4 identical, 4 inline | WordCraft (`crates/doc`, `crates/formats`), craft-libs (`docx`, `wordmodel`) |
| Process-wide warning capture through a `log::Log` impl | 1 | SoundCraft (`crates/clap-host/tests/editor_main_thread.rs:18-30`, a `static WARNINGS: Mutex<Vec<String>>`) |
| Font family renaming for test fonts | 5 | VectorCraft, DesignCraft, WordCraft |
| Pinned corpus lists | Affinity 3 (different contents), PngSuite 2 (different contents), RAF 2 (identical) | PhotoCraft, VectorCraft, craft-libs, LightCraft |

Skip-or-fail variables of one shape (skip a test when something is missing, fail it when a variable
is set) exist for programs, GPUs, corpora, fixtures and craft-fonts: `FILMCRAFT_REQUIRE_ORACLES`,
`PDFCRAFT_ORACLES_REQUIRED`, `CRAFT_PDF_ORACLES_REQUIRED`, `CRAFT_MEDIA_REQUIRE_ORACLES`,
`FILMCRAFT_REQUIRE_GPU`, `EFFECTCRAFT_REQUIRE_GPU`, `PHOTOCRAFT_REQUIRE_GPU_TESTS`,
`PDFCRAFT_TEST_REQUIRE_GPU`, `LIGHTCRAFT_CORPUS_REQUIRED`, `PHOTOCRAFT_CORPUS_STRICT`,
`CADCRAFT_CORPUS_STRICT`, `AFFINITY_CORPUS_REQUIRED`, `CRAFT_AEP_FIXTURES_REQUIRED`,
`CRAFT_CRW_REQUIRE_SAMPLES`, `CRAFT_PRINT_REQUIRE_QUEUE` and `CRAFT_FONTS_REQUIRED`. The copies read
"set" five ways: any value including `0` (`pdfcraft-oracle` `lib.rs:33-39` and its craft-libs port),
`1`, `true`, `yes` or `on` (`craft-media-testkit` `lib.rs:76-77`, `filmcraft-testkit` `oracle.rs:95-96`),
exactly `1` (`craft-fonts-build` `lib.rs:12` and `embed.rs:183`, GridCraft
`crates/ui-egui/src/craft_fonts_policy.rs:58`, FilmCraft `crates/text/build.rs:23`), anything but empty
or `0` (Craft Anything `crates/ui-egui/build.rs:56`), and any value except `0`, empty included
(VectorCraft `crates/affinity/tests/support/corpus.rs:12`). DeckCraft,
FigCraft and WordCraft read `CRAFT_FONTS_REQUIRED` at compile time with `option_env!` (for example
DeckCraft `crates/fonts/src/tests.rs:345-350`), and GridCraft reads it through a constant its build
script writes (`crates/ui-egui/build.rs:52-55`).

### xtask crates

Seventeen Craft repositories have an `xtask/` crate. AniCraft has none, and craft-libs has no xtask,
build-tool or lint crate; outside the Craft repositories only the egui and wgpu forks have one. Open
storytold/craft-libs#9 adds a root `xtask` package to craft-libs.

| Command | Copies | State |
|---|---|---|
| `ci` step runner | 8 plus 2 | Identical from `let mut done = Vec::new();` to the end in CADCraft, DeckCraft, DesignCraft, GridCraft, PhotoCraft, VectorCraft and WordCraft (`xtask/src/main.rs`); craft-launcher differs by 2 lines. MaterialCraft and TextureCraft have their own runner (`cmd_ci`). |
| `layers` | 5 families | The class-based engine is in 9 repositories; from `check()` to `describe()` CADCraft differs from DesignCraft by 1 line, VectorCraft 3, DeckCraft 4, WordCraft 5, LightCraft 8, PhotoCraft 9, PdfCraft 34, GridCraft 45. EffectCraft, FilmCraft, SoundCraft and Craft Anything use a flat table that flags only a dependency on a higher layer. FigCraft allows listed same-layer pairs. MaterialCraft and TextureCraft map crates by folder. AniCraft checks layers in `src/architecture_tests.rs`. |
| `version` | 15 | Blob `8c4c1fc3` in CADCraft, craft-launcher, DeckCraft, DesignCraft, GridCraft, LightCraft and WordCraft; blob `b2f31b20` in MaterialCraft and TextureCraft; 6 other variants. |
| `ico` | 15 | Nine copies differ only in the committed-icon test path, and SoundCraft's also differs in `run`'s signature (`&[String]`) and a doc comment; VectorCraft and Craft Anything share blob `795a792f`; craft-launcher, MaterialCraft and TextureCraft also pack `.icns`. |
| `assets` | 6 plus 2 | CADCraft, DesignCraft, VectorCraft, DeckCraft, WordCraft and GridCraft share the functions; LightCraft and PdfCraft have their own checkers. The extension lists hold 19 to 41 entries. |
| Display-name scanner | 8 plus 1 | `display_names.rs` integration tests in AniCraft, CADCraft, DeckCraft, FigCraft, GridCraft, LightCraft, PhotoCraft and WordCraft; VectorCraft has `cargo xtask brands`. |
| Pinned downloads with SHA-256 | 6 | LightCraft (`sha2`), VectorCraft and PhotoCraft (hand-written `sha256.rs`, 42 diff lines apart), DesignCraft and WordCraft (no hash), craft-libs (`scripts/fetch-affinity-corpus.sh`). |
| `stats` | 8 | Blob `f40b1300` in PhotoCraft and VectorCraft; the other six diverged when each fixed the same directory walk (storytold/craft-libs#138). |
| `pdf-deps` | 4 | VectorCraft, DesignCraft, EffectCraft and DeckCraft pin `craft-pdf` to four different craft-libs revisions: `9be9a637`, `40525913`, `47107f66` and `63196e66`. |

The craft-common-actions `gate.yml` sets up the runner and runs `cargo xtask ci` (`:15-19`), checks
that `Cargo.lock` did not change (`:358-360`), sets `CRAFT_FONTS_DIR` and, when the caller asks,
`CRAFT_FONTS_REQUIRED=1` (`:183-187`). It does not define the gate's steps.

### Shared homes that exist

- `craft-ui` ships `craft_ui::testing` (`Screens`, `ime_events`, `compose`) behind its `screen-testing`
  feature and `docking::testing::DockDriver` behind `testing` (`crates/ui/Cargo.toml:53`, `:56`). No app
  enables either feature. DeckCraft, FigCraft, PhotoCraft, DesignCraft and VectorCraft each build IME
  preedit events with their own `fn preedit`.
- `craft-audio` exports `pcm_wav_file`, `pcm_wav_header` and `wav_float_header` behind `pcm-encode`
  (`crates/audio/src/lib.rs:110-123` at `7ad4765ee5`, and `:101-106` at the revisions FilmCraft,
  EffectCraft, SoundCraft and DeckCraft pin), and all four depend on `craft-audio`.
- `craft-tiff` exposes `writer::IfdBuilder` (`crates/tiff/src/writer.rs:21`), and `craft-psd` exposes
  `builder::PsdBuilder` (`crates/psd/src/builder.rs:212`) with fixtures behind `testgen`. Both exist at
  the revision PhotoCraft pins.
- `craft-pdf` has a port of the `pdfcraft-testkit` `Pdf` builder in `cos/testkit.rs`, declared
  `#[cfg(test)] pub(crate)` (`crates/pdf/src/cos.rs:23-24`), so no other crate can use it.
- `craft-fonts-build` embeds craft-fonts from an app's `build.rs`. Thirteen apps and Craft Anything
  instead read `CRAFT_FONTS_DIR` in their own `build.rs` (`fn craft_fonts` in PhotoCraft
  `crates/text/build.rs:66`, LightCraft `crates/engine/build.rs:64` and twelve more; the full list is the
  `craft-fonts-build` row under "Migrations onto crates that exist"), and the apps reach
  `craft-fonts-build` only through `craft-ui`.
- craft-libs exposes test helpers under seven names today: `test-support` (`craft-zip`, `craft-opc`,
  `craft-wordmodel`, `craft-sf2`), `testgen` (`craft-psd`), `testdata` (`craft-heif`), `test-writer`
  (`craft-pst`), `testing` and `screen-testing` (`craft-ui`), always-public hidden modules
  (`#[doc(hidden)] pub mod testing` in `craft-xlsx` `lib.rs:63-64`, `craft-ods` `lib.rs:33-34` and
  `craft-xls` `lib.rs:49-50`), and a test-only private module (`craft-pdf`).
- `craft-plate-test` holds print-color test scenes in craft-libs, and craft-prepress uses it. It stays
  where it is.

## The family

| Part | Holds | Dependencies | wasm32 |
|---|---|---|---|
| `craft-testkit` | Core modules for every test: folders, skips, programs, fixtures, locks, binaries, deadlines, random input, hostile input, test fonts and source guards | `std`; `tempfile` 3 for `temp` | `rng`, `fonts`, `hostile`'s mutators and `guards` (except `rust_sources`) build for `wasm32-unknown-unknown`; the rest is native only |
| `craft-testkit` features | `image`, `prop`, `corpus`, `mcp`, `log`, `geom` | `png` 0.18, `proptest`, `sha2`, `serde_json`, `log`; `craft-geom` and `kurbo` for `geom` | `image` builds for wasm32 |
| `craft_ui::testing` | Headless egui input, screenshot folders, and the existing IME and dock helpers | egui and egui_kittest, behind `screen-testing` | Native only |
| Format crates' `test-support` | Fixture writers for the formats each crate reads | The crate's own dependencies | As the crate |
| `craft-xtask` | `cargo xtask` command functions configured by each app's policy | `serde_json` for `layers` and `pins` (`cargo metadata`) and `runner`'s JSON report, as in the DesignCraft, VectorCraft, DeckCraft and SoundCraft `xtask/Cargo.toml`; behind the optional `fetch` feature, `sha2` and `craft-testkit` with `corpus`, as LightCraft's `xtask/Cargo.toml` takes `sha2`, with the system `curl` and `tar` that DesignCraft's download runs today. `version`, `icon`, `assets`, `names` and `stats` use `std` only; craft-launcher's `jiff` and `semver` are not used by its `icns.rs`, `ico.rs` or `version.rs` and are not carried | Native only |

The parts depend on each other in one direction:

- `craft-testkit` depends on no craft-libs crate except `craft-geom`, and only through the `geom`
  feature.
- `craft-geom`, `craft-pathops` and `craft-trace` use the `geom` feature only from integration tests
  under `tests/`, never from `#[cfg(test)]` unit tests. A unit test that took it would build a second
  copy of `craft-geom` whose types do not match the crate under test.
- `craft-media-testkit` depends on `craft-testkit`.
- `craft_ui::testing` and the format crates' `test-support` modules do not depend on `craft-testkit`.
- `craft-xtask`'s `fetch` feature depends on `craft-testkit` with the `corpus` feature, for the manifest
  parser that `fetch` shares with `corpus`.

## craft-testkit

Apps and craft-libs crates take `craft-testkit` under `[dev-dependencies]`. `craft-media-testkit` and
`craft-xtask`'s `fetch` feature take it as a normal dependency. An `xtask` may also take it as a normal
dependency, as FilmCraft's does with `filmcraft-testkit` and PdfCraft's with `pdfcraft-oracle`.

### Core modules

| Module | Contents | Comes from | First consumers |
|---|---|---|---|
| `temp` | `TempDir`: a folder made by `tempfile::Builder` with a tag prefix and a random suffix, mode 0o700 and an owner check on Unix, removed with its contents on drop, also when the test panics. A folder it did not make is never emptied or reused. The symlink and owner checks apply to the created folder, not to the components of `std::env::temp_dir()`, and `TempDir::path()` returns the canonicalized path, so a test that compares paths sees `/private/var` on macOS (storytold/vectorcraft#6859). The module's tests include a temp root reached through a symlink. `TempDir::file(name)` joins a file name. | VectorCraft `crates/testkit/src/lib.rs` (checks); `craft-media-testkit` `private_dir()`; CADCraft `tests/support/temp_dir.rs`; the `ScratchDir` of storytold/pdfcraft#6124 | LightCraft, PdfCraft, PhotoCraft, WordCraft, CADCraft, FigCraft, MaterialCraft, DeckCraft, VectorCraft, `craft-media-testkit` |
| `skip` | `skip(reason, required_var)` prints the skip line and returns, or fails the test when the variable is `1`. `or_skip!` binds a value or returns under the same rule. `craft_fonts(present: bool, required: bool)` takes both values from the caller. Today every first consumer passes its own build constant or `option_env!` value (PhotoCraft `crates/text/build.rs:66`, LightCraft `crates/engine/build.rs:64`, DeckCraft `crates/fonts/src/tests.rs:345-350`, GridCraft `crates/ui-egui/build.rs:52-55`), and a repository that switches to `craft-fonts-build` passes the `present` and `required` values from that crate's generated table instead. `craft_fonts_cover(text)` checks coverage as GridCraft's `font_setup.rs:266` does. | LightCraft `lib.rs`; `pdfcraft-oracle` `or_skip!`; FilmCraft `require_*!` macros; `craft-media-testkit` `skip()`; GridCraft `crates/ui-egui/src/theme/font_setup.rs:222-266` | LightCraft, PdfCraft, FilmCraft, DeckCraft, EffectCraft, PhotoCraft, CADCraft, DesignCraft, GridCraft, VectorCraft; craft-libs `aep`, `crw` and `print` tests |
| `oracle` | `find(program, version_arg)`: an explicit path from a variable (a wrong path is an error), then `PATH` with `.exe` on Windows, then known install folders. `find_in(path_var, path_list)` keeps FilmCraft's injectable form for the module's own tests. Results are cached per process with the version. The version output is read up to a byte limit. `ffmpeg_supports_format` and `require_ffprobe!` come along. | `craft-media-testkit` `lib.rs`; FilmCraft `oracle.rs:41`, `:147-181`; `pdfcraft-oracle` `lib.rs:127` | `craft-media-testkit`, `pdfcraft-oracle`, `filmcraft-testkit`, EffectCraft (`crates/opusenc/tests/it/ffmpeg.rs:12-16` and 13 test files that call ffmpeg or ffprobe), SoundCraft (`crates/video/src/tests.rs:327-332`); craft-libs `crates/pdf/tests/cos/oracle.rs` |
| `fixtures` | A collision-free sibling temporary path, publication by rename that reuses a complete concurrent winner, `generate` and `generate_with` under the required-program rule, and `FIXTURE made`, `cached` and `skipped` lines. The root comes from a variable or `<workspace>/target/fixtures`. | FilmCraft `fixtures.rs:56-89` (winner reuse); `craft-media-testkit` `lib.rs:274-289` (required rule) | FilmCraft, EffectCraft (`crates/media/tests/it/footage.rs:19-84`), `craft-media-testkit`; craft-libs `crates/video/tests/vp8/common/mod.rs:75-78` and `crates/audio/tests/codecs/testkit.rs` |
| `serial` | `TestLock`, a const-constructible lock (`static L: TestLock = TestLock::new(); let _g = L.hold();`) that recovers from poisoning and can be held again by the thread that holds it. `gpu_lock()` is one predefined instance, and `hold_gpu_lock()` keeps it until the thread ends. | PhotoCraft `crates/testkit/src/gpu.rs`; DeckCraft `crates/ui-egui/src/gpu_lock.rs`; EffectCraft `crates/ui-egui/tests/it/gpu_test.rs:3-7` and `crates/gpu/src/tests.rs:193-206` (thread-held); PdfCraft `apps/pdfcraft/src/tests.rs:48-53` (poison recovery) | PhotoCraft, DeckCraft, EffectCraft, PdfCraft, WordCraft, LightCraft, AniCraft; the test locks of VectorCraft, CADCraft, FilmCraft and the other repositories in "Helpers copied" |
| `bin` | `bin_path!(name)` expands in the calling test and reads `NEXTEST_BIN_EXE_<name>` with underscores, then with hyphens, then the runtime `CARGO_BIN_EXE_<name>`, then Cargo's compile-time value. A test run from an extracted nextest archive gets the extracted path from `NEXTEST_BIN_EXE_<name>`, and a compile-time path that no longer exists fails the test with a message that names the variables read, instead of OS error 2. The case of storytold/craft-common-actions#225 (storytold/pdfcraft#7173 closed on the same point) is one of the module's tests. `run(path, args, env, max_output) -> Output` runs it and returns an error when stdout or stderr exceeds `max_output` bytes, a limit with a documented default. | SoundCraft `apps/soundcraft-cli/tests/common.rs:7-9` (both spellings); VectorCraft `apps/vectorcraft-cli/tests/it/main.rs:12-15` (runtime lookup) | SoundCraft, VectorCraft, EffectCraft, FilmCraft, LightCraft, PhotoCraft, DeckCraft; the `run` and `cli` helpers in CADCraft, DeckCraft, FilmCraft, GridCraft, PdfCraft, SoundCraft, TextureCraft and WordCraft |
| `deadline` | `with_deadline(limit, label, f)` runs `f` on its own thread and fails the test with the label when the limit passes. `wait_until(limit, what, cond)` polls a condition. Both read an optional limit variable that the caller names. | FilmCraft `deadline.rs:19-23`; `craft-media-testkit` `with_deadline`; PhotoCraft `crates/ui-egui/src/canvas/paint_thread_tests.rs:45` (`wait_until`) | LightCraft (`crates/engine/src/demo/tests.rs:31`), GridCraft (`crates/math/src/lib.rs:675-687`), FilmCraft, Craft Anything (`standin_tools/tests.rs:38`), `craft-media-testkit`; `wait_until` in PhotoCraft, EffectCraft, FilmCraft and LightCraft |
| `rng` | `XorShift64` (`x ^= x << 13; x ^= x >> 7; x ^= x << 17`) and `XorShift64Star`, each seeded, with `next`, `below`, `byte` and `fill`. A suite that switches keeps the generator it uses today, so its inputs do not change. | FilmCraft `crates/dnx/tests/it/fuzz_hostile.rs:9-20` (xorshift64, `next` and `below`) and its craft-libs port `crates/video/tests/dnx/fuzz_hostile.rs`; craft-libs `crates/jpeg/tests/mutation.rs:28-46` (xorshift64\*, with `byte`) | craft-libs `video` (nine `fuzz_hostile.rs` suites) and `jpeg` (`tests/mutation.rs`), FilmCraft, GridCraft, FigCraft |
| `hostile` | `hammer(bytes, parse)`: every truncation, seeded byte flips and extreme-value overwrites, each run under `catch_unwind` and `with_deadline`, with a run-count multiplier read from a variable the caller names. | `craft-media-testkit` `hostile.rs` | CADCraft (`crates/fonts/src/shx/tests.rs:254-272`), TextureCraft (`crates/formats/src/mesh_import/tests.rs:419-435`), craft-libs `mesh` (`src/tests.rs:20`), `craft-media-testkit` |
| `fonts` | `rename_family(bytes, from, to)` for equal-length names in the UTF-16BE and ASCII name records, and `collection(faces)`, which builds a TTC. The caller supplies the font bytes. The source `rename` asserts that the names have the same length (`:66`), and `collection` indexes the table directory without bounds checks (`:92-96`). `rename_family` returns an error for names of different lengths, and `collection` returns an error for a face shorter than its table directory, each with a test. | VectorCraft `crates/text/src/tests/sysfonts.rs:64-101` (`rename` and `collection`) | VectorCraft (`text` and `testkit`), DesignCraft (`crates/fonts/src/fontdb/tests/sysfonts.rs:68`), WordCraft (`crates/fonts/src/fontdb/tests/sysfonts.rs:11-16`, `crates/fonts/src/fontdb/user/tests.rs:6`) |
| `guards` | `predefined_quit`, `wgsl_float_literals`, `std_instant` and `rust_sources`. | storytold/craft-libs#1373, which supersedes storytold/craft-libs#163 and keeps zachshallbetter's commit | PhotoCraft and FilmCraft, which replace their own copies (`crates/gpu/src/tests.rs:233`; `crates/engine/src/wasm_clock_tests.rs:10-46`); EffectCraft (`crates/gpu/src/tests.rs:970`); CADCraft (`predefined_quit` finds `apps/cadcraft/src/native_menu.rs:61`); `rust_sources` in DesignCraft, PhotoCraft, WordCraft and Craft Anything |

### Feature modules

| Module | Contents | Comes from | First consumers |
|---|---|---|---|
| `image` (`png` 0.18) | RGBA8 and gray views; differing pixels, maximum and mean absolute difference, p99 and PSNR; `psnr(a, b, peak)` for u8 and u16 planes; a tolerance with a neighborhood option; a diff image; a bless variable the caller names; failure images in a folder the caller names; and `assert_similar!`, the one form that fails a test. A missing reference fails the test unless the bless variable is set. | FilmCraft `golden.rs`; AniCraft `crates/engine/tests/golden/mod.rs`; `pdfcraft-oracle` `pixels.rs`; VectorCraft `raster.rs` (comparison only); DesignCraft `crates/render/tests/it/effects.rs:47-61`; EffectCraft `crates/export/tests/it/hevc_encoder.rs:57` (u16 PSNR) | FilmCraft, AniCraft, PdfCraft, VectorCraft, DesignCraft; `psnr` in EffectCraft, LightCraft and craft-libs `video`, `image-codecs` and `raster-codecs` tests |
| `prop` (`proptest`) | `config(cases)`, which pins the seed unless `PROPTEST_RNG_SEED` is set, lets `PROPTEST_CASES` override the count and turns failure persistence off. | VectorCraft `crates/pathops/tests/common/mod.rs:11-18` | VectorCraft (`engine` `import_fuzz` and `plugin_fuzz`, `pathops`), craft-libs, LightCraft, FilmCraft, SoundCraft, PhotoCraft, PdfCraft, WordCraft, FigCraft |
| `corpus` (`sha2`) | `pinned(var, manifest) -> Option<PathBuf>`: the folder a variable names, with every listed file checked against its SHA-256, under the skip-or-fail rule. `Manifest` parses the one manifest format of the family: one tab-separated line per file (`sha256`, path, and optional license and URL columns, as in `corpus.tsv`), with `#` comment lines. `craft-xtask`'s `fetch` reads manifests with the same parser. | craft-libs `crates/affinity/tests/support/corpus.rs:18` | craft-libs `affinity` and `raw`, LightCraft (`crates/raw/tests/it/corpus.rs:1185`), PhotoCraft (`crates/io/tests/it/corpus.rs:430`), CADCraft (`crates/io/src/roundtrip/tests.rs:506`), VectorCraft (`crates/affinity/tests/support/corpus.rs`) |
| `mcp` (`serde_json`) | `McpChild::spawn(path, args, env)`, `initialize(version)`, `notify`, `request(method, params)` matched by id, `call_tool(name, args)` and `close` with the exit status. Each read is bounded by `deadline`, and each line by a length limit. | craft-ai-integrations `crates/ai-mcp/src/apps/child.rs` (`Conn`, `request`, `call_tool`, `MAX_LINE`, deadlines) and its 14 tests in `child_tests.rs`; VectorCraft `apps/vectorcraft-cli/tests/it/mcp_e2e.rs:77-84`; PhotoCraft `apps/photocraft-cli/tests/it/cli.rs:1224-1230` | The 12 repositories in "Helpers copied"; all of them send protocol version `2025-06-18` today. craft-ai-integrations keeps `child.rs`, which is production code, and the `mcp` pull request ports the cases of `child_tests.rs` to `craft-testkit` |
| `log` (`log`) | `warnings(f) -> (T, Vec<String>)` and `records(level, f)`, captured per thread, and `capture_global(level)`, which installs one logger for the whole process and returns a handle whose `drain()` takes the records logged on every thread. `log::set_logger` can be called once per process, so one test binary uses either the per-thread capture or `capture_global`, and the second to be installed returns an error. | `test_log.rs` (WordCraft `crates/doc/src/test_log.rs`, identical in `crates/formats` and craft-libs `docx` and `wordmodel`); SoundCraft `crates/clap-host/tests/editor_main_thread.rs:18-41` (process-wide) | WordCraft, craft-libs `docx` and `wordmodel`, and SoundCraft through `capture_global`: `editor_main_thread.rs` is a `harness = false` binary that installs its own logger (`:41`) and checks a warning logged on a spawned `audio` thread (`:99-101`) |
| `geom` (`craft-geom`, `kurbo`, `proptest`) | Proptest strategies for paths and shapes, and geometry assertions such as Hausdorff closeness. | VectorCraft `crates/testkit/src/strategies.rs` and `crates/testkit/src/geom.rs`; RFC-0030's `crates/testkit` (wdunn001/craft-libs branch `shared-geometry-crates`, `799a7d8604`: `geom.rs` and `strategies.rs`) | VectorCraft and FigCraft, which RFC-0030 names as consumers; the tests of RFC-0030's `craft-geom`, `craft-pathops` and `craft-trace` |

### guards

storytold/craft-libs#139 proposes source guards for three platform rules that several apps broke.
zachshallbetter's storytold/craft-libs#163 implements them as `crates/testkit`, package `craft-testkit`.
storytold/craft-libs#1373 supersedes #163: it keeps his commit with its authorship and adds fixes after
it. The guards are:

- `predefined_quit(files)` lists calls of muda's `PredefinedMenuItem::quit` outside comments
  (storytold/designcraft#544, storytold/filmcraft#1943). `craft_quit::QUIT_MENU_ID`
  (`crates/quit/src/close.rs:20-26`) replaces the call.
- `wgsl_float_literals` and `wgsl_float_literals_in` list WGSL decimal and hexadecimal float literals
  outside `f32`'s finite range, and literals with an `h` suffix outside `f16`'s, from PhotoCraft's
  scanner (storytold/photocraft#1982, storytold/photocraft#2023).
- `std_instant(files)` lists lines that name `std::time::Instant` outside comments, with FilmCraft's
  rule, and accepts lines under a non-wasm `cfg` (storytold/filmcraft#230). GridCraft enforces the same
  rule with clippy (`clippy.toml:9-11`) and keeps that configuration.
- `rust_sources(dir)` lists a crate's `.rs` files without its test files and build output.

Each app calls the guards from one test and keeps its file lists, its exceptions and its clippy
configuration. Rules that ast-grep can express can also run through the craft-common-actions
`actions/ast-grep` action.

### deadline after the LightCraft stack

storytold/lightcraft#3083 adds `crates/testkit/src/deadline.rs` (`time_scale`, `scaled`, `wait_until`,
`wait_until_at_least`, `recv_within`, `LIGHTCRAFT_TIME_SCALE`) and `watchdog.rs`, which prints the tests
still running, exits with code 124 and calls `std::process::abort` 10 seconds later. It has no
`with_deadline`. storytold/lightcraft#3096, #3657, #3722 and #3733 build on it. All four are drafts, and
#3657, #3722 and #3733 conflict with `main`. #3083 adds `deadline.rs` with 243 lines, and at the head of
#3733, which carries the whole stack, the file has 298 lines. After
the stack merges, `deadline` takes `time_scale`, `scaled`, `recv_within` and the watchdog from the
merged result, and `with_deadline` and `wait_until` start scaling their limits by a time-scale variable
the caller names.

### Variables and printed lines

Each function that reads an environment variable takes the variable's name from the caller and has a
`CRAFT_` default. `prop::config` reads proptest's own `PROPTEST_RNG_SEED` and `PROPTEST_CASES`, and
`bin_path!` reads the `NEXTEST_BIN_EXE_<name>` and `CARGO_BIN_EXE_<name>` variables that nextest and
Cargo set. Each app passes the names it uses today, such as `FILMCRAFT_REQUIRE_ORACLES`,
`FILMCRAFT_BLESS`, `FILMCRAFT_HOSTILE_TIMEOUT_SECS`, `PDFCRAFT_ORACLES_REQUIRED`, `UPDATE_GOLDENS`,
`DESIGNCRAFT_BLESS`, `CRAFT_MEDIA_REQUIRE_ORACLES`, `CRAFT_MEDIA_HOSTILE_MULT` and
`CRAFT_TEST_TIMEOUT_SECS`. The printed lines that an xtask reads keep their format: FilmCraft's
`cargo xtask fixtures` reads the `FIXTURE` lines, and LightCraft's `xtask/src/ci_report.rs` reads the
skip log that `lightcraft-testkit` keeps writing on top of `skip`.

A required variable counts as set only when its value is exactly `1`, as `craft-fonts-build` reads
`CRAFT_FONTS_REQUIRED` and as `gate.yml` sets it. That reading changes behavior in `pdfcraft-oracle`
and its craft-libs port (any value), `filmcraft-testkit` and `craft-media-testkit` (`1`, `true`, `yes`
or `on`), VectorCraft's Affinity corpus (any value except `0`), Craft Anything's `build.rs` (anything
but empty or `0`), and the `build.rs` readers that test `var_os("CRAFT_FONTS_REQUIRED").is_some()`:
DeckCraft `crates/fonts/build.rs:53`, LightCraft `crates/engine/build.rs:16`, PdfCraft
`crates/fonts/build.rs:21`, VectorCraft `crates/text/build.rs:27`, FigCraft `crates/text/build.rs:25`,
PhotoCraft `crates/text/build.rs:30`, DesignCraft `crates/fonts/build.rs:16`, WordCraft
`crates/fonts/build.rs:23` and `:29`, CADCraft `crates/fonts/build.rs:21` and TextureCraft
`crates/ui-egui/build.rs:35`. Test and xtask code reads variables the same way: `CRAFT_FONTS_REQUIRED`
in PhotoCraft `crates/ui-egui/tests/it/camera_raw_localization.rs:132`, DesignCraft
`crates/fonts/src/fontdb/scoped.rs:462` and `crates/ui-egui/src/canvas/tests.rs:2889`, and LightCraft
`xtask/src/web.rs:56`, and `PHOTOCRAFT_CORPUS_STRICT` in PhotoCraft `crates/io/tests/it/corpus.rs:430`.
FilmCraft's `crates/text/build.rs:23` already reads exactly `1`. Each of these repositories changes its
reading in its own pull request and checks that its workflows set the variable to `1`.

craft-testkit has no helper that changes the process environment. No test in the Craft repositories
calls `std::env::set_var` (the one call, VectorCraft `apps/vectorcraft/src/cursor_theme.rs:142`, is app
code). A test that needs a variable runs the program as a child, and `bin`, `mcp` and `fixtures` take
an environment list for that child.

## UI test support in craft-ui

`craft_ui::testing` holds the test support that needs egui. It gains two modules behind `screen-testing`:

- `input`: `move_to`, `press`, `release`, `click`, `drag(from, to, steps, modifiers)`,
  `key(key, modifiers)`, `type_text` and `wheel`, generic over `Harness<State>`. They are filled from
  PhotoCraft's `crates/ui-egui/src/canvas/paint_mouse/tests.rs:42` and EffectCraft's
  `crates/ui-egui/tests/it/ui_text_edit.rs:33-59`. `DockDriver` drives its own egui `Context` with
  `RawInput` and has no `Harness`, so its `window_click` and `window_hold`
  (`crates/ui/src/docking/testing.rs:247-256`) stay as they are. First consumers: PhotoCraft and
  EffectCraft, then GridCraft.
- `snapshot`: `snapshot_dir(var) -> Option<PathBuf>`, a builder step that turns wgpu on only when a
  folder is set, and `save(&mut Harness, dir, name)`. The caller names the variable, and
  `CRAFT_UI_SNAPSHOT_DIR` is the default. The module writes screenshots for review and compares
  nothing. It takes its code from FilmCraft `crates/ui-egui/tests/it/export_ui.rs:34-40` (the variable
  and the wgpu step) and `:110-115` (the save), and from the `snapshots` functions that read
  `CRAFT_UI_PANES_SNAPSHOT_DIR` in FilmCraft `crates/ui-egui/tests/it/ui_dock_resize.rs:22`, `:53-57`
  and EffectCraft `crates/ui-egui/tests/it/ui_dock_resize.rs:13`, `:42-45` (`e4ab3c6a99`,
  `f029bbadb2`). FilmCraft (`FILMCRAFT_UI_SNAPSHOT_DIR` in 25 files) and EffectCraft (`EC_SNAPSHOT_DIR`
  and 19 other names) are its first consumers.

Apps switch their `fn preedit` builders to `ime_events` and `compose`, and their dock tests to
`DockDriver`. Every app except PdfCraft, AniCraft and Craft Anything patches egui_kittest to
storytold/egui rev `6d518fc799`. PdfCraft pins rev `725a39fca9` (`Cargo.toml:183`), AniCraft takes
`egui = "0.36"` from crates.io and has no egui_kittest dependency (`Cargo.toml:59`), and Craft Anything
takes `egui = "0.36"` and `egui_kittest = "0.36"` from crates.io with no storytold/egui patch
(`Cargo.toml:40-42`). Each of the three moves to the shared revision before it takes the `input`
module.
More UI test helpers for accessibility follow RFC-0014 (storytold/craft-contrib#21) and
storytold/craft-libs#437 in the same module.

## Test fixtures in the format crates

Every craft-libs crate that exposes test-only fixture writers puts them in `pub mod testing` under
`cfg(any(test, feature = "test-support"))`. Writers that are part of a crate's public API, such as
`craft_tiff::writer`, `craft_psd::builder` and `craft-audio`'s `pcm-encode` exports, keep their paths
and features. The move:

- `craft-pdf` exposes `cos::testkit::Pdf` as `craft_pdf::testing::Pdf` behind `test-support`.
  `pdfcraft-testkit` re-exports it and deletes its own builder. VectorCraft's and DesignCraft's
  object-level PDF fixtures move to it. Fixtures that need RC4 encryption (VectorCraft `pdf.rs`) or a
  broken cross-reference table stay in the apps until the builder has them.
- `craft-psd` renames `testgen`, `craft-heif` renames `testdata` and `craft-pst` renames `test-writer` to
  `test-support`. Each keeps the old name as an alias until every repository that enables it has
  switched: PhotoCraft enables `craft-psd`'s `testgen` (`crates/automation/Cargo.toml:59`,
  `crates/io/Cargo.toml:71`, `crates/ui-egui/Cargo.toml:80`), LightCraft enables `craft-heif`'s
  `testdata` (`crates/codecs/Cargo.toml:52`, `crates/engine/Cargo.toml:69`, `crates/meta/Cargo.toml:21`),
  craft-libs `crates/psd/Cargo.toml:29` enables `testgen` for the crate's own tests, and no repository
  enables `craft-pst`'s `test-writer`. A later pull request per crate removes the alias. Each crate's
  tests pass with the old name and with `test-support`. `craft-xlsx`, `craft-ods` and `craft-xls` gate
  their `testing` modules (`crates/xlsx/src/lib.rs:63-64`, `crates/ods/src/lib.rs:33-34`,
  `crates/xls/src/lib.rs:49-50`) under `cfg(any(test, feature = "test-support"))`, the form
  `gridcraft-xlsx` uses (`crates/xlsx/src/lib.rs:35-36`, `crates/xlsx/Cargo.toml:14` at `c2aba7a3f4`).
  Their integration tests under `tests/` call `testing` today, so each crate enables the feature for
  them through a dev-dependency on itself, as `craft-psd` does for `testgen`. `craft-ui` keeps
  `testing` and `screen-testing` as the names for UI drivers.
- A well-formed TIFF or PSD fixture is built with `craft_tiff::writer` or `craft_psd::builder`.
  PhotoCraft's orientation and EXIF headers (`apps/photocraft-cli/tests/it/orientation.rs`,
  `crates/io/tests/it/orientation.rs`, `crates/engine/tests/it/jpeg_exif_resolution.rs`) and
  VectorCraft's `crates/testkit/src/tiff.rs` (except `group3()`, its CCITT Group 3 case) move first.
- A plain PCM WAV fixture calls `craft_audio::pcm_wav_file`, and a float fixture uses
  `craft_audio::wav_float_header`, with `pcm-encode` under `[dev-dependencies]`. FilmCraft's
  `testkit/src/wav.rs` and the copies in EffectCraft, DeckCraft and SoundCraft move. Writers for
  malformed or extensible headers (FilmCraft `crates/interchange/tests/it/wav_extensible.rs`,
  EffectCraft `crates/host/tests/it/wav_markers.rs`) stay local.
- A format's pinned corpus list moves to the crate that reads the format: the Affinity list to
  `craft-affinity`, which already holds `tests/support/corpus.tsv`; the RAF list to `craft-raw`, which
  already holds the same blob as LightCraft (`0fd9572836`); and PngSuite to the PNG decoder's crate.
  PhotoCraft's, VectorCraft's and `craft-affinity`'s Affinity lists (60, 53 and 53 entries) and
  PhotoCraft's and VectorCraft's PngSuite lists (178 and 177 entries) are merged into one list each. The merged list is the union of the entries. Every list is written in the
  `Manifest` format: `corpus.tsv` already has it, and the RAF list (`crates/raw/docs/raf-corpus.sha256`)
  and the PngSuite lists (`xtask/pngsuite.sha256`), which use `sha256sum`'s `<sha256>  <path>` lines,
  are converted by replacing the two spaces with a tab, with their `#` lines kept. An entry whose name
  appears with two different SHA-256 values stays out of the list until its source file is checked,
  and the pull request lists each such conflict. Before an app switches to a merged list, its corpus
  fetch downloads every entry of that list, so a corpus that holds all of the app's current entries is
  not reported as incomplete because the union names more files. Each app's switch reports its corpus
  tests' pass and skip counts before and after with the merged list.

## craft-xtask

`craft-xtask` is a library. Each app keeps its `xtask` binary, its step list and its app-only commands,
and calls the library with its own policy. Every layers table classifies `craft-xtask` as build
tooling, allowed only as a dependency of `xtask`.

| Module | Contents | Comes from | First consumers |
|---|---|---|---|
| `runner` | `Step { name, run }`, `run_steps` with the shared ok, FAIL and skip summary, `root`, `cargo`, `run(Command, label)` and `nextest_or_test`, which falls back to `cargo test` when nextest is absent. An opt-in lock held across processes and a JSON report are features. On Windows a running `xtask.exe` cannot be replaced while the workspace tests rebuild it (storytold/craft-common-actions#185), so the test steps exclude the `xtask` package from the tested workspace, and a separate step runs the `xtask` package's own tests with a `--target-dir` of their own, which does not replace the running executable. A test checks that the step list never rebuilds the running executable. | DesignCraft `xtask/src/main.rs:69-79`, `:204-247`; LightCraft `xtask/src/ci_lock.rs` and `ci_report.rs` | CADCraft, DeckCraft, DesignCraft, GridCraft, PhotoCraft, VectorCraft, WordCraft, craft-launcher; then MaterialCraft and TextureCraft |
| `layers` | The class-based engine configured by a `Policy` (crate table, same-layer rule `Allowed` or `Ordered`, UI crates and their lowest layer, exceptions, format pairs), and `wasm_set(policy)`. PdfCraft's sideways and standalone exceptions, PhotoCraft's `standalone_exception` and GridCraft's `FORMAT_ALLOWED` become policy fields. | DesignCraft `xtask/src/layers.rs:151` onward (CADCraft differs by 1 line) | CADCraft, DeckCraft, DesignCraft, WordCraft, VectorCraft; then the flat-table repositories with `same_layer: Allowed`; AniCraft, through the small xtask that "Faults in the current copies" adds, which replaces `src/architecture_tests.rs` |
| `version` | `cargo xtask version [set X.Y.Z]`, rejecting numeric parts that overflow u64 and numeric prerelease identifiers with a leading zero, moving the `version` of path dependencies in `[workspace.dependencies]`, and restoring `Cargo.toml` when `cargo update --workspace` fails. | SoundCraft `xtask/src/version.rs:51-121`; PdfCraft `xtask/src/version.rs:31-42`, test `:146` | All 15 repositories with a copy |
| `icon` | `ico::pack`, which refuses empty input, non-square images and sides of 0 or over 256 px; `icns::pack`; `png_size`. The committed-icon test takes the app's icon path. | DeckCraft `xtask/src/ico.rs` (same as CADCraft `:21-31`); craft-launcher `xtask/src/icns.rs` | The 15 repositories with a copy |
| `assets` | Every asset file needs a backticked row in `ASSETS.md`. One extension list (the union of the seven lists, plus `ttc` and `otc`) with a per-app `extra_ext`, and LightCraft's forbidden extensions and names as an option. | VectorCraft or DesignCraft `xtask/src/assets.rs`; LightCraft `xtask/src/assets.rs` | CADCraft, DesignCraft, VectorCraft, WordCraft, DeckCraft, GridCraft, LightCraft |
| `names` | A scanner of production string literals for display names a caller lists, with escapes decoded, plus VectorCraft's extra surfaces (`Cargo.toml` descriptions, `packaging/`, non-Markdown `docs/`, the `brand-ok` marker). Each repository's `display_names.rs` test calls it with its own `BANNED` and `ALLOWED` lists. RFC-0029 (storytold/craft-contrib#43) is the rule it enforces. | CADCraft `crates/ui-egui/tests/display_names.rs:109-115`; VectorCraft `xtask/src/brands.rs` | AniCraft, CADCraft, DeckCraft, FigCraft, GridCraft, LightCraft, PhotoCraft, WordCraft, VectorCraft |
| `fetch` | `fetch_pinned(dest, &[Pin { name, url, sha256, max_bytes }])`, which deletes a file whose hash does not match, bounded archive extraction, and manifests read with `craft_testkit::corpus::Manifest`. It uses `sha2`, and VectorCraft's and PhotoCraft's `sha256.rs` files are deleted. | LightCraft `xtask/src/corpus.rs:71-93`; DesignCraft `xtask/src/main.rs:251-350` (`expands_within`); VectorCraft `xtask/src/pngsuite.rs:10-134` | LightCraft, VectorCraft, PhotoCraft, DesignCraft, WordCraft |
| `stats` | Tests and lines of Rust per crate, without following directory links. Test items used only by the unix symlink test are imported under `cfg(unix)`, and craft-libs `workspace.yml` runs clippy on Windows (`:47-51`). | DesignCraft `xtask/src/stats.rs:21-23` | The 8 repositories with a copy |
| `pins` | `check(metadata, &[Pin { name, version, source, allowed_features, forbidden_packages }])` over `cargo metadata`. Each app keeps its table. | DeckCraft `xtask/src/pdf_deps.rs:10-78` and the other three copies | VectorCraft, DesignCraft, EffectCraft, DeckCraft |

craft-common-actions keeps runner setup, caching, nextest installation, the lockfile check and the
`git-sources` source-prefix check. `craft-xtask` holds what `cargo xtask ci` runs, locally and in
`gate.yml` alike.

## Rules for every crate

- **Dependency kind.** Apps and craft-libs crates take `craft-testkit`, `craft-ui`'s test features and
  the format crates' `test-support` under `[dev-dependencies]`. `craft-media-testkit`, `craft-xtask`'s
  `fetch` feature and an app's `xtask` take `craft-testkit` as a normal dependency. `craft-xtask` is a
  dependency of `xtask` only. An app with a testkit class in `xtask/src/layers.rs` adds `craft-testkit` to it in its adoption
  pull request; GridCraft, FigCraft, MaterialCraft and TextureCraft add the class (or take
  `craft_xtask::layers`) in theirs. DesignCraft, DeckCraft, CADCraft and WordCraft keep their testkit
  class.
- **Panics.** Setup functions return a `Result`. Failing a test happens only in macros that expand in
  the caller (`or_skip!`, `require_ffprobe!`, `assert_similar!`, `bin_path!`) and in `with_deadline`,
  `wait_until`, `skip` with a required variable, `skip::craft_fonts` with a required value,
  `corpus::pinned`, `fixtures::generate` and `generate_with` under the required rule, and
  `hostile::hammer`, each of which carries a documented `#[allow(clippy::panic)]`.
  The crate root otherwise copies the `deny` attribute of Rust house rule 1
  (`standards/house-rules-rust.md`).
- **Clock.** No helper returns elapsed time for a test to assert on. `deadline` uses the clock only to
  stop work that does not finish. Tests of the crates use handshakes, and randomized tests use a fixed
  seed.
- **Limits.** Every input read from disk, from a download or from a child process has a size limit that
  is a parameter with a documented default: the width, height and byte count of a PNG in `image`, the
  line length in `mcp`, the output size in `bin::run` and in `oracle`'s version probe, and `max_bytes`
  and the entry count in `fetch`. An input over a limit returns an
  error, as `craft_mesh::Limits` does.
- **Network.** No module of `craft-testkit` opens a connection. Downloads are made by `craft-xtask`'s
  `fetch`.
- **Lints.** `#![forbid(unsafe_code)]` and `[lints] workspace = true`, as `craft-media-testkit`'s
  manifest has.
- **Licenses.** Every dependency is on the house-rules license allowlist as it stands. `tempfile`,
  `png`, `proptest`, `sha2`, `serde_json` and `log` are already dependencies of craft-libs crates or of
  the apps. `craft-xtask` takes `serde_json` and, through `fetch`, `sha2`, which the apps' `xtask`
  manifests already name. If open RFC-0027 (storytold/craft-contrib#41) is accepted, the rule follows
  its allowlist.
- **Admission.** Each module's README row names at least two Craft applications that consume it, as
  craft-libs `AGENTS.md:61` requires and as open RFC-0009 (storytold/craft-contrib#10) proposes. Each
  crate's README has an Origin table (source repository, path and SHA of each module's code) and a
  Migration table (each consumer and the state of its switch), and names a responsible maintainer or
  team, agreed in the review of its first craft-libs pull request.
- **One revision.** Each adoption pull request moves all of the repository's craft-libs git
  dependencies to the one revision that holds the new code, so `Cargo.lock` holds one copy of each
  craft-libs crate. Every git dependency comes from a storytold repository; the `git-sources` job in
  craft-common-actions `release.yml` fails on any other git source.
- **CI.** craft-libs `workspace.yml` runs clippy and the tests of every crate with and without
  `--all-features` (`:42-53`). A new crate adds no workflow. The parts of `craft-testkit` that the
  family table marks native only (`temp`, `skip`, `oracle`, `fixtures`, `serial`, `bin`, `deadline`,
  `hostile::hammer`, `guards::rust_sources`, and the `prop`, `log`, `geom`, `corpus` and `mcp`
  features) are under
  `cfg(not(target_arch = "wasm32"))`. Its wasm32 check is a row in craft-libs `formats.yml` that builds
  the default features plus `image`, added by a separate pull request (house rule 11). `craft-xtask`
  has no wasm32 row. craft-libs `ui.yml` tests `craft-ui` with `--all-features` (`:55`), which covers
  `craft_ui::testing`.

## Faults in the current copies

The shared code holds the fixed form of each of these. Each repository takes the fix in its own pull
request with a test that fails without it, as item 3 of "How the code moves" says, before or after its
switch.

- 12 of the 15 `version set` copies accept a prerelease that Cargo rejects, such as `1.0.0-01` and
  `1.0.0-rc.01` (CADCraft `xtask/src/version.rs:51-52` and the copies with the same blob). Only
  VectorCraft (`:146`), PhotoCraft (`:153`) and SoundCraft (`:189`) test it. Open
  storytold/pdfcraft#6188 (MMexicann) fixes PdfCraft's copy, and the count becomes 11 of 15 when it
  merges. Open storytold/photocraft#10189 (Lucenx9) makes PhotoCraft's copy reject version parts that
  overflow. 14 copies leave `Cargo.toml` changed when the following `cargo update --workspace` fails;
  SoundCraft (`:114-121`) restores it.
- VectorCraft's and Craft Anything's `ico::pack` accepts non-square images and images with a 0 px side,
  and it writes a 0 px width as 0, which the `.ico` format reads as 256 px (VectorCraft
  `xtask/src/ico.rs:26-31`).
- The display-name scanners in LightCraft (`crates/ui-egui/tests/display_names.rs:97`), FigCraft
  (`crates/mcp/tests/display_names.rs:97`) and PhotoCraft (`crates/ui-egui/tests/display_names.rs:117`)
  read the `"` of `'\"'` as the start of a string, so a banned name after such a literal is not seen.
- WordCraft's `cargo xtask corpus --download` fetches PngSuite over plain HTTP and extracts it without a
  size limit, an entry limit or a hash check (`xtask/src/main.rs:268-297`). DesignCraft fetches the same
  URL with limits and no hash check.
- `cargo xtask stats` follows directory links in PhotoCraft and VectorCraft (`xtask/src/stats.rs:21`)
  and GridCraft (`:45`).
- The `stats` tests in WordCraft (`xtask/src/stats.rs:113`) and GridCraft (`:146`) import `walk` and
  `Counts` for all targets, while only the unix symlink test uses them, so
  `cargo clippy --all-targets -- -D warnings` fails on Windows. Open pull requests fix it one copy at a
  time: storytold/wordcraft#6247 (NibrasAz7) and #6235 (dkhlapov), and storytold/gridcraft#1803
  (derechopiano), which carries `#[cfg(unix)] use super::{Counts, walk};`. CADCraft fixed the same
  import on `main` in storytold/cadcraft#2198, and smacdmac101's open storytold/cadcraft#2288, #2281,
  #2277, #2271 and #2231 carry that fix as well.
- LightCraft's `cargo xtask` help says `corpus --download` fetches PngSuite (`xtask/src/main.rs:66-67`),
  and `cmd_corpus` downloads only the raw samples and the D750 ColorChecker set
  (`xtask/src/corpus.rs:37-66`).
- `holders_take_turns` in PhotoCraft (`crates/testkit/src/gpu.rs`) and DeckCraft
  (`crates/ui-egui/src/gpu_lock.rs`) sleeps 2 ms. `craft-media-testkit`'s `hostile::run_one` asserts
  each run takes under 5 s (`hostile.rs:27-33`). `serial` and `hostile` replace these with a handshake
  and `with_deadline`.
- `pdfcraft-oracle` says its any-value rule matches `CRAFT_FONTS_REQUIRED` (`lib.rs:33-39`), and
  `craft-fonts-build` reads only `1` (`lib.rs:12`).
- VectorCraft turns proptest failure persistence off and still has
  `crates/pathops/proptest-regressions/planar/tests.txt`. craft-libs has ten committed proptest
  regression files, LightCraft has four, SoundCraft has three, and PhotoCraft, DesignCraft, FilmCraft and
  PdfCraft have six more. Under `prop`, a failing case becomes a
  named regression test, and each adoption pull request turns the repository's committed cases into
  tests.
- AniCraft's gate runs `cargo test` and clippy only (`.github/workflows/ci.yml:31`). No AniCraft
  workflow runs `cargo fmt --check`, and AniCraft has no `ASSETS.md` while `assets/app-icon/` holds PNG,
  SVG and `.ico` files. A small AniCraft xtask on `craft_xtask::runner`, `assets` and `layers` adds these
  steps.

## How the code moves

Each move is made in three kinds of pull request:

1. A craft-libs pull request adds the code with the tests of the code it comes from.
2. One pull request per repository switches it to the shared code and deletes its copy. The tests the
   repository keeps pass and skip as before, each moved test passes in craft-libs under the same name,
   and the pull request lists both sets with their counts. The variable names the repository uses do
   not change. An xtask switch shows that `cargo xtask ci` runs the same steps with the same results,
   and an `ico` switch compares the written `.ico` with `cmp`.
3. A change of behavior, such as a required variable's reading, CADCraft's `TempDir` (which empties a
   leftover folder of the same name; `temp` leaves it in place and makes a new one), DesignCraft's
   golden that writes a missing reference and passes, a seeded suite that changes generator, or a fault
   listed above, is its own pull request. Each such pull request includes a test that fails without the
   change and lists the tests or gate steps whose result changes. A switch of a flat-table repository
   to `layers` lists every new violation the shared engine reports, with an exception or a fix for
   each. A change to a required variable's reading names each workflow that now fails instead of
   skipping. A switch to the shared `assets` list lists every file that the union list newly flags,
   with the `ASSETS.md` row or the exclusion it adds for each. A switch to `prop::config` names each
   suite whose seed changes from random to pinned, and keeps a variable or a CI step that still runs
   that suite with random seeds. A switch to `names` lists every literal that the shared scanner newly
   flags, with the `ALLOWED` entry or the fix for each. A switch to `stats` reports the per-crate counts
   before and after.

The craft-libs pull requests go in this order:

1. `craft-testkit` with `guards`, from storytold/craft-libs#1373. It supersedes
   storytold/craft-libs#163, which conflicts with `main`, as FredApps noted in a review comment on
   2026-11-01. #1373 keeps zachshallbetter's commit from #163 with his authorship.
2. The core modules, one pull request each: `temp`, `skip`, `serial`, `bin`, `deadline` (`with_deadline`
   and `wait_until`), `oracle`, `fixtures`, `rng`, `hostile`, `fonts`.
3. The feature modules: `image`, `prop`, `log`, `corpus`, `mcp`.
4. `craft_ui::testing::input` and `snapshot`.
5. The format crates' `test-support` renames and `craft-pdf`'s `testing::Pdf`, one crate per pull request.
6. `craft-xtask`, one module per pull request, `runner`, `layers`, `version` and `icon` first.
7. `craft-media-testkit` built on `craft-testkit`, with its public names unchanged and `hostile`
   re-exported. This pull request changes code the media reviewers checked in storytold/craft-libs#701,
   and its author requests reviews from alehel and FredApps. craft-media, craft-cfb, craft-audio and
   craft-video report unchanged pass and skip counts, with `CRAFT_MEDIA_REQUIRE_ORACLES=1` where their
   workflow sets it (`media.yml:83`, `audio.yml:162`). Among the testkit folders, the path filters of
   `media.yml`, `audio.yml` and `video.yml` name only `crates/media-testkit/**` (`:6`, `:8` in each).
   They also match `Cargo.toml` and `Cargo.lock`, so the pull request that adds `crates/testkit`
   starts all three, but a later change to `crates/testkit/**` alone does not. The `oracle` pull
   request adds `crates/testkit/**` to those path filters.
8. `deadline`'s time scale and watchdog, after the LightCraft stack merges.
9. The `geom` feature, after RFC-0030 (storytold/craft-contrib#46) is accepted.

The repository pull requests start with the near-identical copies: CADCraft's two `TempDir` files, the
repositories with the identical `ci` runner or `version.rs`, and DesignCraft and DeckCraft for
`layers`. FilmCraft's switch also moves `crates/codecs/src/tests.rs:10-28` and
`crates/ui-egui/examples/bench/fixtures.rs:15-18` onto `oracle` and `fixtures`. VectorCraft's switch
includes the two `TempFolder` copies in `crates/mcp/src/tests.rs:51` and
`crates/mcp/tests/common/mod.rs:9-28` and the three `TestDir(tempfile::TempDir)` wrappers. PhotoCraft's
includes `crates/gpu/tests/it/main.rs:5`.

Where a repository still carries its own copy of a craft-libs crate, its test suite has diverged from
the crate's. Before such a copy is deleted, the repository's tests and `tests/support` are compared
with the crate's, and every case the crate lacks is ported to the crate first. The known pairs are
WordCraft `crates/docx/tests/it` and craft-libs `crates/docx/tests/it` (50 shared file names, none
identical), PdfCraft `crates/cos/tests/it/proptests.rs` and craft-libs
`crates/pdf/tests/cos/proptests.rs`, LightCraft `crates/meta` and `crates/raw` and craft-libs
`metadata` and `raw`, FilmCraft `crates/dnx/tests/it/fuzz_hostile.rs` and craft-libs
`crates/video/tests/dnx/fuzz_hostile.rs`, and AniCraft and craft-libs `crates/xfl/tests/support/fla.rs`.

## What stays in the apps

- **App fixtures and models.** VectorCraft's document fixtures, invariants, `render_*` helpers, `ai.rs`,
  `ase.rs`, `suitcase_fork` and `group3()`; PhotoCraft's `Sweep`, `perf` and the wgpu instance that
  storytold/photocraft#10865 keeps alive; FilmCraft's `cuts.rs` and the golden `.attribution` sidecar;
  LightCraft's skip log, its `Need` enum and `crates/ui-egui/src/tests/fixtures.rs`; PdfCraft's XFA
  fixtures and the `ScratchFile` of open storytold/pdfcraft#6124; FigCraft's server sessions and
  `.sketch` fixtures; EffectCraft's `.aep` helpers and QA harness; AniCraft's `.fla` packer;
  WordCraft's `crates/doc/src/work.rs`; SoundCraft's CLAP and VST3 fixture plugins; Craft Anything's
  samples; and fixtures that are malformed on purpose.
- **App xtask commands.** The step list of each `cargo xtask ci`, the checklist commands that RFC-0018
  (storytold/craft-contrib#30) and RFC-0033 (storytold/craft-contrib#52) cover, `i18n-report`, whose
  missing-translation report can move to craft-i18n's `craft-i18n-tests` or craft-libs `i18n`,
  LightCraft's `ci --quick`, EffectCraft's `ci --affected` and PdfCraft's license checker.
- **Text snapshots.** `FILMCRAFT_BLESS` text fingerprints, GridCraft's `NUMFMT_BLESS` and PdfCraft's
  `UPDATE_SNAPSHOTS` stay in their repositories.
- **Private temporary folders in shipped code.** PdfCraft `crates/ocr/src/tesseract.rs:519` and Craft
  Anything `crates/engine/src/tools/java_decompile_tools.rs:58` and `crates/engines/src/ghidra/mod.rs:413`
  cannot use a test-support crate. A non-test home for them, such as `craft-atomic`, is a candidate below.

## Candidates for later RFCs

The apps also duplicate runtime code for which craft-libs has no crate. Each item lists its copies
(repository and path, at the SHAs in "What exists"), the copy whose file and test files hold the most
`#[test]` functions, two first consumers, and the open pull requests in its area. Each item can become
its own crate RFC in the shape of this one. This RFC does not decide any of them. Runtime code that
craft-libs already has a crate for is under "Migrations onto crates that exist", which this RFC
decides.

| Candidate | Copies | Source (tests) | First consumers |
|---|---|---|---|
| App control channel | `apps/<app>/src/control_server.rs` in VectorCraft, PhotoCraft, DesignCraft, LightCraft, FilmCraft, EffectCraft, GridCraft, WordCraft, DeckCraft, SoundCraft, CADCraft, FigCraft, MaterialCraft and TextureCraft | FilmCraft (22) | WordCraft (19) and CADCraft (13) |
| MCP stdio server | `crates/mcp/src/server.rs` in VectorCraft, DesignCraft, LightCraft, GridCraft, WordCraft, DeckCraft, CADCraft and FigCraft; EffectCraft `crates/automation/src/server.rs`; PdfCraft and SoundCraft `crates/automation/src/mcp.rs`; AniCraft `crates/cli/src/mcp.rs`; Craft Anything `crates/mcp/src/lib.rs`; craft-ai-integrations `crates/ai-mcp/src/mcp.rs` | PdfCraft (6) | SoundCraft and AniCraft |
| Browser downloads in web builds | `web.rs` under `apps/<app>-web/src/` in VectorCraft, PhotoCraft, DesignCraft, LightCraft (also `safety.rs`), GridCraft, WordCraft, DeckCraft, SoundCraft and FigCraft; FilmCraft `apps/filmcraft-web/src/fs.rs`; EffectCraft `apps/effectcraft-web/src/files.rs`; PdfCraft `crates/ui-egui/src/editing/save.rs` and `workflows/web_print.rs`; CADCraft `apps/cadcraft-web/src/download.rs`; AniCraft `src/web.rs`; MaterialCraft and TextureCraft `crates/ui-egui/src/web_files.rs` | MaterialCraft (7) | DeckCraft and SoundCraft |
| macOS open-documents handler | `apps/<app>/src/apple_events.rs` in PhotoCraft, DesignCraft, EffectCraft, PdfCraft, GridCraft, WordCraft, DeckCraft and FigCraft; VectorCraft `apps/vectorcraft/src/open_documents.rs` | FigCraft (1, in `apple_events_tests.rs`) and VectorCraft (1) | PhotoCraft and DesignCraft |
| Single instance | `apps/<app>/src/single_instance.rs` in PdfCraft, PhotoCraft and VectorCraft; craft-launcher `crates/native/src/single_instance.rs` | PhotoCraft (15) | PdfCraft (11) and VectorCraft (10) |
| Window graphics startup | VectorCraft `apps/vectorcraft/src/gpu.rs`; PhotoCraft `apps/photocraft/src/gpu_startup.rs`; DesignCraft `apps/designcraft/src/gpu.rs`; LightCraft `apps/lightcraft/src/window_gpu.rs`; `apps/<app>/src/graphics.rs` in FilmCraft, GridCraft, WordCraft, DeckCraft, SoundCraft, CADCraft and FigCraft; EffectCraft and PdfCraft `apps/<app>/src/main.rs`; AniCraft `src/startup.rs`; TextureCraft `apps/texturecraft/src/gpu.rs`; craft-launcher `apps/artcraft-launcher/src/main.rs`; Craft Anything `apps/craft-anything/src/main.rs` | VectorCraft (39) | CADCraft and DeckCraft |
| Document autosave and recovery | VectorCraft `crates/ui-egui/src/recovery.rs`; PhotoCraft `crates/format/src/autosave.rs`; `crates/engine/src/recovery.rs` in DesignCraft, WordCraft, DeckCraft and AniCraft; `crates/engine/src/autosave.rs` in FilmCraft, SoundCraft and CADCraft; EffectCraft `crates/engine/src/settings/autosave.rs`; PdfCraft `crates/ui-egui/src/persist/recovery.rs`; FigCraft `apps/figcraft/src/recovery.rs`; TextureCraft `crates/formats/src/project_file/recovery.rs` | DesignCraft (84) | DeckCraft and WordCraft |
| WebAssembly plug-in host | VectorCraft and PhotoCraft `crates/plugins`, AniCraft `crates/plugins`, EffectCraft and FigCraft `crates/plugin` | VectorCraft (172) | PhotoCraft and AniCraft |
| Recent files | VectorCraft `crates/ui-egui/src/io/open.rs`; PhotoCraft and DeckCraft `crates/ui-egui/src/app/files.rs`; DesignCraft `crates/engine/src/lib.rs` (`MAX_RECENT_FILES`); FilmCraft `crates/engine/src/settings/schema.rs`; EffectCraft `crates/engine/src/settings/prefs/recent.rs`; PdfCraft `crates/ui-egui/src/state.rs`; GridCraft `crates/ui-egui/src/backstage.rs`; WordCraft `crates/ui-egui/src/lib.rs`; SoundCraft `crates/ui-egui/src/dialogs/recent.rs`; CADCraft `crates/ui-egui/src/lib.rs`; FigCraft `crates/engine/src/cmd/file/open.rs`; AniCraft `crates/engine/src/cmds_recent.rs`; MaterialCraft `crates/ui-egui/src/state/ui_state.rs` | AniCraft (5, in a module of its own) | SoundCraft and FigCraft |
| Display ICC profile | PhotoCraft `apps/photocraft/src/monitor_profile.rs`; VectorCraft `apps/vectorcraft/src/display_profile.rs`; LightCraft `apps/lightcraft/src/screen_profile.rs` and `win_screen_profile.rs` | PhotoCraft (7) | VectorCraft and LightCraft |
| UI clock | `crates/ui-egui/src/clock.rs` in LightCraft and EffectCraft (`AppClock` with a manual override), and in VectorCraft, DesignCraft, GridCraft, WordCraft, DeckCraft and FigCraft (monotonic milliseconds only) | EffectCraft (no tests in the file) | LightCraft and DesignCraft |
| Private temporary folders in shipped code | PdfCraft `crates/ocr/src/tesseract.rs`; Craft Anything `crates/engine/src/tools/java_decompile_tools.rs` and `crates/engines/src/ghidra/mod.rs` | PdfCraft (24) | PdfCraft and Craft Anything |

- **App control channel.** Each copy is a loopback listener the MCP process talks to, with bounded line
  reads, a request queue to the UI thread and a reply timeout (228 to 707 lines). Seven authenticate
  with a per-run token or key file (CADCraft, FilmCraft, GridCraft, MaterialCraft, PhotoCraft,
  TextureCraft, WordCraft), and seven accept any local connection on the port (DesignCraft, EffectCraft,
  DeckCraft, FigCraft, LightCraft, SoundCraft, VectorCraft). DesignCraft's and EffectCraft's copies
  differ in 15 lines with the same eight tests. craft-libs has no crate. The `crates/ipc` crate of open
  storytold/craft-libs#616 (see "Single instance") has a server (`src/server/`), a client
  (`src/client.rs`) and `src/auth.rs`.
- **MCP stdio server.** Thirteen apps hand-write initialize with version negotiation, `tools/list` and
  `tools/call`; all thirteen support `2025-06-18`, `2025-03-26` and `2024-11-05`. VectorCraft also
  negotiates `2026-07-28` in `initialize` (`crates/mcp/src/server.rs:22-23`), and LightCraft
  (`crates/mcp/src/server.rs:248`) and EffectCraft (`crates/automation/src/server.rs:281-285`) accept
  clients that declare `2026-07-28` per request in `_meta`.
  PhotoCraft, FilmCraft, MaterialCraft and TextureCraft use `rmcp`. MaterialCraft's and TextureCraft's
  `budgets.rs`, `bridge.rs` and `security.rs` are near copies of PhotoCraft's. craft-ai-integrations
  `crates/ai-mcp/src/mcp.rs` (`e8b31cb059`) is a fourteenth hand-written server; it supports
  `2025-06-18` and `2025-11-25` (`:13`) and none of the older versions. craft-libs has no crate. Open
  storytold/craft-libs#248 (Jeremy8776, draft) plans native MCP readiness for the apps, and open RFC-0016
  (storytold/craft-contrib#23) proposes a hosted connector with its own MCP endpoint.
- **Browser downloads in web builds.** Each copy makes a Blob, an object URL, a hidden `<a download>`
  and a timed revoke, with revoke delays of 5, 10 or 60 seconds and its own MIME table. DeckCraft's and
  SoundCraft's copies differ only in the MIME table. `craft-web-host` has a host-save hook and no
  download helper. Open RFC-0023 (storytold/craft-contrib#39) proposes a web runtime for the apps, and
  this candidate is left to it.
- **macOS open-documents handler.** The nine apps queue the files that Finder sends in an
  open-documents event.
  `craft-platform` has no handler.
- **Single instance.** PdfCraft, PhotoCraft and VectorCraft pass a second launch's files to the running
  app, and craft-launcher tells the running copy to show its window (533 to 751 lines each). craft-libs
  has no crate. Open storytold/craft-libs#616 (bigwolfeman, CONFLICTING) adds a `crates/ipc` crate with
  `src/activate.rs` and `src/auth.rs`, which also bears on the app control channel.
- **Window graphics startup.** Each copy sets up wgpu for the app window. CADCraft's, DeckCraft's and
  SoundCraft's `graphics.rs` are near copies. `craft-window` does not use wgpu.
- **Document autosave and recovery.** `craft-settings` `recovery.rs` keeps only damaged settings files.
  Open storytold/craft-libs#713 (dooreelko) lets a web host report that it stores saves in place
  (`crates/web-host`).
- **WebAssembly plug-in host.** Each copy builds a wasmi 2 host with fuel metering; PhotoCraft's engine
  configuration line appears in AniCraft unchanged. None of the five uses the craft-extensions
  `craft-ext-core` manifest crate. craft-libs `main` has no host. Open craft-libs pull requests by
  bigwolfeman add one, and all of them conflict with `main`: storytold/craft-libs#432
  (`craft-plugin-abi`), #454 (`craft-plugin-host` engines: wasmi, with optional wasmtime and browser
  engines), #455 (the guest SDK), #613 (the host layer), #615 (`craft-plugin-ui`) and #616 (`craft-cli`,
  with `crates/ipc`).
- **Recent files.** The lists have different caps (10 in SoundCraft and AniCraft, 20 in FigCraft, 30 in
  VectorCraft, 50 in WordCraft, up to 100 in DesignCraft) and different missing-file checks. craft-libs
  has no crate.
- **Display ICC profile.** The copies look up the profile of the display that shows the window.
  `craft-cms` has no lookup.
- **UI clock.** `craft-time` holds media time only.
- **Private temporary folders in shipped code.** These are the folders listed under "What stays in the
  apps". `craft-atomic` exists and holds file writes only.

The per-app geometry crates are not candidates. MaterialCraft's and TextureCraft's `crates/geom` are
one forked crate, and RFC-0032 (storytold/craft-contrib#50) proposes moving it into `craft-mesh-ops`.
The other twelve apps' `crates/geom` are separate domain models (VectorCraft's and DesignCraft's share
only `snap.rs`) and stay in the apps; a four-point homography solve exists in VectorCraft, LightCraft,
PdfCraft and PhotoCraft.

## Migrations onto crates that exist

The apps also keep their own copies of runtime code that a craft-libs crate already holds. This RFC
decides that each copy in the table below moves onto its crate. No new crate is needed. Each move is
made in the three kinds of pull request of "How the code moves":

1. A craft-libs pull request adds to the crate every case from the source of missing cases that the
   crate's tests lack, with the case's original test name.
2. One pull request per repository switches it to the crate and deletes its copy. The repository's
   tests pass and skip as before, and the pull request lists both counts.
3. A change of behavior that the switch brings, such as a different permission, retry or folder list,
   is its own pull request with a test that fails without it.

| Crate | Repositories that use it | Copies it replaces | Source of missing cases (tests) | First consumers |
|---|---|---|---|---|
| `craft-quit` | EffectCraft, PdfCraft, MaterialCraft, TextureCraft, craft-launcher | Their own `on_exit`: VectorCraft `apps/vectorcraft/src/app.rs:100`; PhotoCraft `crates/ui-egui/src/app.rs:250`; DesignCraft `apps/designcraft/src/main.rs:107`; LightCraft `apps/lightcraft/src/main.rs:185`; FilmCraft `crates/ui-egui/src/app.rs:507`; GridCraft `apps/gridcraft/src/main.rs:64`; WordCraft `apps/wordcraft/src/main.rs:131`; DeckCraft `apps/deckcraft/src/main.rs:52`; SoundCraft `apps/soundcraft/src/main.rs:103`; CADCraft `apps/cadcraft/src/main.rs:127`; FigCraft `apps/figcraft/src/main.rs:132`; AniCraft `src/app.rs:3656`; Craft Anything `crates/ui-egui/src/lib.rs:2233` | The crate's own tests | DeckCraft and CADCraft |
| `craft-dialogs` | DesignCraft, MaterialCraft, TextureCraft | Direct `rfd` dependencies: VectorCraft `apps/vectorcraft/Cargo.toml:31`; PhotoCraft `apps/photocraft/Cargo.toml:47`; LightCraft `apps/lightcraft/Cargo.toml:44`; FilmCraft `apps/filmcraft/Cargo.toml:35`; EffectCraft `apps/effectcraft/Cargo.toml:24`; PdfCraft `crates/ui-egui/Cargo.toml:47`; GridCraft `apps/gridcraft/Cargo.toml:40`; WordCraft `apps/wordcraft/Cargo.toml:90`; DeckCraft `apps/deckcraft/Cargo.toml:31`; SoundCraft `apps/soundcraft/Cargo.toml:38`; CADCraft `apps/cadcraft/Cargo.toml:45`; FigCraft `apps/figcraft/Cargo.toml:31`; AniCraft `Cargo.toml:54`; craft-launcher `apps/artcraft-launcher/Cargo.toml:65`; Craft Anything `apps/craft-anything/Cargo.toml:23` | EffectCraft `apps/effectcraft/src/file_dialogs.rs` (5) and WordCraft `apps/wordcraft/src/file_dialogs.rs` (5) | EffectCraft and WordCraft |
| `craft-atomic` | DesignCraft (`Cargo.toml:99`, `crates/engine/Cargo.toml:49`; `crates/engine/src/atomic.rs:1-18` wraps `craft_atomic::atomic_write` and `atomic_write_with`) | VectorCraft `crates/format/src/atomic.rs`; PhotoCraft `crates/format/src/atomic.rs`; LightCraft `crates/catalog/src/safe_file.rs`; FilmCraft `crates/format/src/atomic.rs`; EffectCraft `crates/engine/src/settings/config.rs`; PdfCraft `crates/platform/src/staging.rs`; DeckCraft `crates/ui-egui/src/settings.rs` and `crates/engine/src/recovery.rs`; SoundCraft `crates/engine/src/io.rs`; CADCraft `crates/engine/src/atomic_write.rs`; FigCraft `crates/format/src/lib.rs`; AniCraft `crates/engine/src/recovery.rs`; MaterialCraft `crates/formats/src/file.rs`; TextureCraft `crates/formats/src/lib.rs`; craft-launcher `crates/native/src/paths.rs`; Craft Anything `crates/platform/src/fs.rs`; and further private copies in the same repositories | PhotoCraft `crates/format/src/atomic.rs` (39) | VectorCraft and FilmCraft |
| `craft-font-catalog` | WordCraft (`Cargo.toml:33`, `crates/engine/Cargo.toml:46`, used at `crates/engine/src/packs/store.rs:7`); FigCraft (`Cargo.toml:50`, `crates/engine/Cargo.toml:40`, `crates/text/Cargo.toml:31`; `crates/text/src/fontdb.rs:1330` takes the Linux and BSD font folders from `craft_font_catalog::font_dirs`) | Installed-font discovery: VectorCraft `crates/text/src/fontdb/dirs.rs`; PhotoCraft and FilmCraft `crates/text/src/fonts.rs`; DesignCraft `crates/fonts/src/fontdb/system.rs`; EffectCraft `crates/text/src/font_dirs.rs`; PdfCraft `crates/render/src/system_font.rs`; GridCraft `crates/ui-egui/src/fonts.rs`; WordCraft `crates/fonts/src/fontdb/names.rs:210` (`system_font_dirs`); DeckCraft `crates/fonts/src/fontdb.rs`; CADCraft `crates/fonts/src/ttf.rs`; FigCraft's macOS and Windows folders in `crates/text/src/fontdb.rs:1300-1325`; TextureCraft `crates/formats/src/fonts.rs` | FigCraft `crates/text/src/fontdb_tests.rs` (30), for the macOS and Windows cases | DeckCraft and WordCraft (`crates/fonts`) |
| `craft-menu` | PdfCraft, FigCraft, MaterialCraft, TextureCraft | Direct `muda` dependencies: VectorCraft `apps/vectorcraft/Cargo.toml:71`; PhotoCraft `apps/photocraft/Cargo.toml:71`; DesignCraft `apps/designcraft/Cargo.toml:48`; LightCraft `apps/lightcraft/Cargo.toml:50`; FilmCraft `apps/filmcraft/Cargo.toml:51`; EffectCraft `apps/effectcraft/Cargo.toml:51`; GridCraft `apps/gridcraft/Cargo.toml:54`; SoundCraft `apps/soundcraft/Cargo.toml:45`; CADCraft `apps/cadcraft/Cargo.toml:52`; Craft Anything `crates/ui-egui/Cargo.toml:42` | The crate's own tests | CADCraft and GridCraft |
| `craft-i18n` | Every app through `craft-ui`; PdfCraft, PhotoCraft, WordCraft and TextureCraft also from their UI crate | TSV catalogs with an app `tl!` macro: VectorCraft `crates/ui-egui/src/lib.rs:19`, `crates/ui-egui/src/i18n.rs` and 28 catalogs in `crates/ui-egui/src/i18n/`; CADCraft `crates/ui-egui/src/i18n.rs:433` and 18 catalogs; FilmCraft `crates/ui-egui/src/lib.rs:20` and 16 catalogs; craft-launcher `crates/ui-egui/src/lib.rs:13` and 12 catalogs in `crates/ui-egui/src/i18n/`. Rust tables: `crates/ui-egui/src/i18n.rs` in DesignCraft, GridCraft, EffectCraft, LightCraft, FigCraft and MaterialCraft | VectorCraft `crates/ui-egui/src/i18n/tests.rs` (122) | CADCraft and FilmCraft |
| `craft-cms` | PhotoCraft, DesignCraft, PdfCraft | ICC code: VectorCraft `crates/color/src/cms.rs` and `crates/color/src/cms/` (`icc.rs`, `display.rs`, `generic.rs`, `lab.rs`); EffectCraft `crates/color/src/icc.rs`; FigCraft `crates/color/src/icc.rs` (a profile writer); TextureCraft `crates/formats/src/icc.rs`; LightCraft `crates/codecs/src/icc.rs` | VectorCraft `crates/color/src/cms/tests.rs` and `tests_icc.rs` (40) | EffectCraft (7) and LightCraft (6) |
| `craft-settings` | FigCraft (`Cargo.toml:57`, `apps/figcraft/Cargo.toml:24`) | Settings stores: VectorCraft `apps/vectorcraft/src/prefs.rs`; LightCraft `apps/lightcraft/src/prefs.rs` (`ui.json`, with damaged files set aside at `:79`); EffectCraft `crates/engine/src/settings/config.rs`; DeckCraft `crates/ui-egui/src/settings.rs`; AniCraft `src/prefs.rs` | VectorCraft `apps/vectorcraft/src/prefs.rs` (16) | DeckCraft and LightCraft |
| `craft-fonts-build` | Every app through `craft-ui` only | `build.rs` readers of `CRAFT_FONTS_DIR`: VectorCraft `crates/text/build.rs:79`; PhotoCraft `crates/text/build.rs:66`; DesignCraft `crates/fonts/build.rs:30`; LightCraft `crates/engine/build.rs:64`; FilmCraft `crates/text/build.rs:56`; PdfCraft `crates/fonts/build.rs:41`; GridCraft `crates/ui-egui/build.rs:78`; WordCraft `crates/fonts/build.rs:47`; DeckCraft `crates/fonts/build.rs:83`; CADCraft `crates/fonts/build.rs:41`; FigCraft `crates/text/build.rs:131`; TextureCraft `crates/ui-egui/build.rs:49`; Craft Anything `crates/ui-egui/build.rs:83` (each `fn craft_fonts`); AniCraft `build.rs:38` (`discover`) | GridCraft `crates/ui-egui/src/craft_fonts_policy.rs` (8) | DesignCraft and CADCraft |
| `craft-xlsx` | DesignCraft | GridCraft `crates/xlsx` (`gridcraft-xlsx`, `Cargo.toml:22`), which shares nine file names with craft-libs `crates/xlsx/src` (`lib.rs`, `drawing.rs`, `package.rs`, `pivot.rs`, `richdata.rs`, `styles.rs`, `testing.rs`, `write.rs`, `xlsb.rs`) and depends on `craft-xls` (`Cargo.toml:43`) but not on `craft-xlsx` | `gridcraft-xlsx`'s `crates/xlsx/src/tests.rs` and `crates/xlsx/src/tests/` | GridCraft and `craft-xlsx` |
| `craft-raw` | PhotoCraft, LightCraft | LightCraft `crates/raw` (`lightcraft-raw`, `Cargo.toml:45`), which depends on `craft-raw` (`crates/raw/Cargo.toml:13`) since storytold/lightcraft#4244 removed its `crates/raw/src/vendor/` decoders, and keeps 7 files with the same names as files in `craft-raw`'s `crates/raw/src` (5 modules besides `lib.rs` and `tests.rs`: `dngwrite.rs`, `gaintable.rs`, `opcodes.rs`, `profile.rs` and `semantic.rs`) | LightCraft `crates/raw/tests/` (21 files) | LightCraft and `craft-raw` |
| `craft-metadata` | LightCraft | LightCraft `crates/meta` (`lightcraft-meta`, `Cargo.toml:47`), which depends on `craft-metadata` (`crates/meta/Cargo.toml:13`) since storytold/lightcraft#4244 and keeps `exif.rs`, `focus.rs`, `iptc.rs` and `xmp.rs`, the same file names as in craft-libs `crates/metadata/src` | LightCraft `crates/meta/tests/` and the modules' own tests | LightCraft and `craft-metadata` |
| `craft-video` (`dnx` feature) | FilmCraft, EffectCraft, DeckCraft, SoundCraft | FilmCraft `crates/dnx` (`filmcraft-dnx`, `Cargo.toml:22`), whose eight modules have the same file names as craft-libs `crates/video/src/dnx/` (`bits.rs`, `dct.rs`, `decode.rs`, `encode.rs`, `ffmpeg_probe.rs`, `header.rs`, `spec_tables.rs`, `tables.rs`) | FilmCraft `crates/dnx/tests/it` (7 files) | FilmCraft and `craft-video` |
| `craft-xfl` | AniCraft (`Cargo.toml:35`, `crates/xfl/Cargo.toml:16`) | AniCraft `crates/xfl/src/edges.rs`, the edge-string parser that craft-libs `crates/xfl/src/edges.rs` also holds | AniCraft `crates/xfl/src/tests.rs` and craft-libs `crates/xfl/tests/support/fla.rs` | AniCraft and `craft-xfl` |

- **craft-quit.** VectorCraft, MaterialCraft and EffectCraft each have an unsaved-changes prompt.
- **craft-dialogs.** EffectCraft's guarded `apps/effectcraft/src/file_dialogs.rs` is the closest to it.
- **craft-atomic, then craft-settings.** The copies of a temporary-file, flush and rename write differ in
  Windows retries, symlink handling, permissions and folder sync. DesignCraft already writes through
  `craft-atomic`, and `craft-settings` is used by FigCraft only. craft-launcher
  `crates/native/src/paths.rs:132`, CADCraft `crates/engine/src/atomic_write.rs:282` and `:320`, and
  CADCraft `apps/cadcraft-cli/src/lib.rs:171` are the tests `craft-atomic` passes before a repository
  switches.
- **craft-font-catalog and craft-ui's `UiFonts`.** WordCraft uses `craft-font-catalog` in
  `crates/engine` and keeps its own folder list in `crates/fonts`. FigCraft takes its Linux and BSD
  folders from the crate and keeps its macOS and Windows folders. No app uses
  `craft_ui::fonts::UiFonts`; DeckCraft, MaterialCraft, TextureCraft and VectorCraft define their own
  types of that name. VectorCraft's
  `crates/text/src/fontconfig.rs` and `suitcase.rs` have the same module names as
  `craft-font-catalog`'s.
- **craft-i18n.** VectorCraft, CADCraft, FilmCraft and craft-launcher keep TSV catalogs with their own
  `tl!` macro, and DesignCraft, GridCraft, EffectCraft, LightCraft, FigCraft and MaterialCraft keep Rust
  tables. Open RFC-0011 (storytold/craft-contrib#15) moves language data to craft-i18n, and these
  switches follow it if it is accepted.
- **craft-cms.** `craft-cms`, VectorCraft and FigCraft use moxcms 0.8. LightCraft's `Cargo.toml` names
  moxcms 0.9 (`:94`), and its lock holds only 0.8.1.
- **craft-fonts-build.** The fourteen `build.rs` readers in the table switch to it with the embedded font
  list unchanged, and each then passes the values from the crate's table to `skip::craft_fonts`.
- **Format crates that an app also keeps.** The `craft-xlsx`, `craft-raw`, `craft-metadata`,
  `craft-video` and `craft-xfl` rows follow the porting rule in "How the code moves": the app's tests
  are compared with the crate's, every case the crate lacks is ported to the crate first, and the app
  then switches and deletes its copy. WordCraft's `wordcraft-docx` (`crates/docx/src/lib.rs` delegates to
  `craft-docx`) and PdfCraft's `pdfcraft-cos` (`crates/cos/src/lib.rs` is `pub use craft_pdf::cos::*;`)
  already use their craft-libs crates, so only their test suites move.

Open craft-libs pull requests change these crates, and each would change what an app's switch is
tested against. A switch onto one of these crates merges with the pull requests on that crate or waits
for them, as in "Defaults this RFC takes", item 12:

- `craft-atomic` and `craft-settings`: storytold/craft-libs#558 (dylanmtaylor) changes
  `crates/atomic/src/lib.rs` and `crates/settings/src/recovery.rs` to give a replaced file's group bits
  only to its group, which bears on the permissions in which the copies differ.
  storytold/craft-libs#576 (CypherNaught-0x, draft) adds transactional updates to `craft-settings`, and
  #515 (SDG-Den, draft) adds a settings overlay and a shared palette.
- `craft-font-catalog`: aiongg's drafts storytold/craft-libs#376, #381, #382, #383, #385 and #386 rework
  `crates/font-catalog` (document font scopes, an app module, a face cache, a font store and
  manager), and #377 (aiongg, draft) adds a `craft-ui` system font fallback over the catalog. All seven
  conflict with `main`.
- `craft-menu`: storytold/craft-libs#501 (ahmedhosnypro, draft) mirrors the menu bar for right-to-left
  languages.

## Defaults this RFC takes

1. **Names.** RFC-0030 (storytold/craft-contrib#46) proposes a crate also named `craft-testkit` for
   geometry assertions and proptest strategies (wdunn001/craft-libs branch `shared-geometry-crates`,
   `crates/testkit`, depending on `craft-geom`, `kurbo` and `proptest`). This RFC adds those helpers as a
   `geom` module of the same crate behind a `geom` feature. Whichever RFC's crate lands first creates
   `craft-testkit`, and the other adds its modules to it. `craft-xtask` is the name
   storytold/craft-libs#138 uses.
2. **One crate with features.** `craft-testkit` is one crate, and a module with a dependency sits behind
   a feature.
3. **craft-media-testkit.** It stays, with its public names. storytold/craft-libs#701 removed an earlier
   `craft-testkit` that only wrapped it (alehel,
   <https://github.com/storytold/craft-libs/pull/701#issuecomment-6721885995>). Each module of this
   `craft-testkit` holds moved code with its tests, and `craft-media-testkit` is rebuilt on it in its own
   later pull request.
4. **UI support.** Test helpers that need egui go in `craft_ui::testing`, and `craft-testkit` has no egui
   dependency. UI screenshots use egui_kittest's own snapshots or `snapshot`, and `image` adds no second
   UI snapshot mechanism.
5. **GPU.** `serial` orders tests on real devices and does not replace `craft-gpu-null`, which RFC-0022
   requires every GPU consumer to run against in CI. `craft-testkit` has no wgpu dependency.
6. **Temporary folders.** `temp` wraps `tempfile`, which FilmCraft's testkit and storytold/filmcraft#4681
   already use, and adds the mode and owner checks. RFC-0015's `confine` crate keeps its escape tests and
   may use `temp` in them.
7. **Required variables.** A required variable is set when its value is exactly `1`.
8. **Testkit layer class.** The class stays in DesignCraft, DeckCraft, CADCraft and WordCraft, and
   `craft-testkit` fills it. This settles the question storytold/designcraft#1980 leaves open.
9. **storytold/craft-libs#115.** Its testkit entry names a corpus manifest, large-file fixtures and a
   mutation-fuzz helper. They map to `corpus` (from `craft-affinity`'s loader; storytold/filmcraft#429
   closed unmerged), `fixtures` and `hostile`.
10. **Host-supplied fonts.** Open RFC-0034 (storytold/craft-contrib#54) asks every build that adopts it
    to pass acceptance tests "from shared fixtures", and it changes what "craft-fonts present" means for
    wasm32 builds. Its shared font fixtures go in `craft-testkit::fonts` or in a font crate's
    `test-support`, and the values that native test runs pass to `skip::craft_fonts` follow it if it is
    accepted.
11. **Out of scope.** Font, CFF, Type 1 and ICC builders in open pull requests (storytold/pdfcraft#6733,
    #6840, #6977, #7281, #2777, #3028; storytold/deckcraft#874; storytold/cadcraft#1587) wait until one
    merges and a second app has a merged copy. The four different `craft-pdf` revisions in the `pins`
    tables are for the PDF crate's maintainers to settle.
12. **Other contributors' work.** This RFC changes no open pull request. storytold/pdfcraft#6124,
    #5923, storytold/filmcraft#4681, storytold/photocraft#13266 and storytold/texturecraft#254 make test
    folders private, and each repository switches to `temp` after its pull request merges.
    storytold/photocraft#10865, storytold/filmcraft#3805 and storytold/lightcraft#3519 change GPU test
    serialization or skip logging. storytold/filmcraft#2835 and #1952 change FilmCraft's oracle
    code, and FilmCraft's switch merges with them or waits. storytold/filmcraft#1794 stays open as its
    author left it. storytold/designcraft#1984 and #2000 are left as they are. The LightCraft stack from
    storytold/lightcraft#3083 is the source of `deadline`'s later part.
    storytold/craft-libs#504 (el-j, a draft that conflicts with `main`) adds a `crates/testkit` package
    named `craft-testkit` whose `lib.rs` re-exports `craft_media_testkit` and adds `skip(what, reason)`,
    and a `.github/workflows/testkit.yml` that runs fmt, clippy, the tests and a wasm32 check on Ubuntu
    and Windows, with ffmpeg installed on Ubuntu. Its wrapper and `testkit.yml` are not carried forward:
    `craft-testkit` comes from storytold/craft-libs#1373 (which supersedes #163) and the modules this
    RFC lists, and #504's video
    crates switch to `craft-testkit` after it lands. Open app pull requests also change source files
    this RFC moves:
    - storytold/photocraft#14595 (davidmann4, draft) makes `gpu_lock()` in `crates/testkit/src/gpu.rs`
      keep a wgpu Vulkan instance on Linux, which adds a wgpu dependency. PhotoCraft's switch keeps the
      kept instance in `photocraft-testkit` around `serial::gpu_lock`, as it does for
      storytold/photocraft#10865.
    - storytold/lightcraft#3710 (THEtheChad, draft) rewrites `xtask/src/ci.rs` (+144/-22), which runs
      LightCraft's gate steps with `ci_lock` and `ci_report`, the sources of `runner`'s lock held across
      processes and its JSON report.
    - storytold/wordcraft#6247 (NibrasAz7) changes `xtask/src/stats.rs`.
    - storytold/vectorcraft#1437 (EricKuck) adds `cargo xtask cleanroom` and changes
      `xtask/src/brands.rs`, the source of `names`.
    - storytold/vectorcraft#7203 (dotCooCoo) changes `crates/testkit/src/strategies.rs`, the source of
      `geom`.
    - storytold/lightcraft#3086 (THEtheChad, draft) changes `xtask/src/ci.rs`, `xtask/src/test.rs` and
      `.config/nextest.toml`, and storytold/lightcraft#3079 (THEtheChad) changes `xtask/src/ci.rs` and
      `xtask/src/test.rs`. Both change LightCraft's test caps, which `runner` takes in.
    - storytold/filmcraft#4083 (vangel76) takes `NVENC_SERIAL` in every NVENC test, a consumer of
      `serial`.
    - storytold/designcraft#2549 (ForestParks) changes DesignCraft's MCP test fixtures in
      `apps/designcraft-cli/tests/it/cli.rs`, a consumer of `mcp`.
    - storytold/pdfcraft#6188 (MMexicann) makes PdfCraft's `xtask/src/version.rs` reject numeric
      prerelease identifiers with a leading zero, and storytold/photocraft#10189 (Lucenx9, conflicting)
      makes PhotoCraft's reject version parts that overflow. Both are existing fixes for faults that
      `version` takes in. storytold/gridcraft#2784 (Harrison-Booth) and storytold/filmcraft#4615
      (Neo-DannyDeTour) also change `xtask/src/version.rs`.
    - storytold/pdfcraft#4764 (Harrison-Booth, conflicting) changes `crates/oracle/src/lib.rs` and its
      README, the source of `oracle`.
    - storytold/lightcraft#3099 and #3102 (THEtheChad, drafts) change `xtask/src/ci_lock.rs`, the source
      of `runner`'s lock held across processes, and `xtask/src/ci.rs`. storytold/lightcraft#3654
      (THEtheChad, draft), #3118 (THEtheChad) and #4167 (dylanmtaylor, conflicting) also change
      `xtask/src/ci.rs`.
    - storytold/lightcraft#3112 and #3110 (sm18lr88, conflicting) change `xtask/src/corpus.rs`, the source
      of `fetch` (`:71-93`).
    - storytold/effectcraft#3643, #3646, #3649, #3653 and #3671 (dylanmtaylor, drafts) change
      `xtask/src/pdf_deps.rs`, a source of `pins`.
    - storytold/vectorcraft#817 (Haven-King) changes `crates/testkit/src/strategies.rs`, the source of
      `geom`.
    - storytold/wordcraft#6235 (dkhlapov) and storytold/gridcraft#1803 (derechopiano) change the `stats`
      tests, as #6247 does.

    The `runner`, `stats` and `names` switches in LightCraft, WordCraft and VectorCraft merge with these
    pull requests or wait for them, LightCraft's `runner` switch does the same for #3086, #3079, #3099,
    #3102, #3654, #3118 and #4167, and `geom` takes `strategies.rs` as it stands after #7203 and #817.
    PdfCraft's and PhotoCraft's `version` switches merge with storytold/pdfcraft#6188 and
    storytold/photocraft#10189 or wait for them, and the `version` pull request in craft-libs credits
    both as the existing fixes. GridCraft's and FilmCraft's `version` switches do the same for #2784 and
    #4615, PdfCraft's `oracle` switch for #4764, LightCraft's `fetch` switch for #3112 and #3110,
    EffectCraft's `pins` switch for its five `pdf_deps.rs` pull requests, and WordCraft's and
    GridCraft's `stats` switches for #6235 and #1803.

    Open craft-libs pull requests also change files that this RFC moves or gates:
    - storytold/craft-libs#1112 (dylanmtaylor) rewrites `media.yml`, `audio.yml`, `video.yml` and
      `formats.yml`, which step 7 of the craft-libs order gives new path filters and which get the
      `craft-testkit` wasm32 row.
    - storytold/craft-libs#1309 (regendorf-ux) and #1277 (DesignsByRay) change `crates/pdf/src/cos.rs`,
      where `cos::testkit` is declared (`:23-24`).
    - storytold/craft-libs#1337, #1330, #1323, #1263 and #1166 (keithcsqueensu) change
      `crates/xlsx/src/lib.rs`, whose `testing` module (`:63-64`) this RFC gates.
    - storytold/craft-libs#1321 (dylanmtaylor) changes the cfg block of `crates/audio/src/lib.rs` next to
      the `pcm-encode` exports.
    - storytold/craft-libs#172 (adventurelands) moves `craft-ui` to egui 0.37, while the apps patch
      egui_kittest to storytold/egui rev `6d518fc799`.
    - storytold/craft-libs#9 (zachshallbetter) adds a root `xtask` package to craft-libs
      (`members = ["crates/*", "xtask"]`) with its own `xtask/src/ci.rs` and `cargo xtask compat`.
      `craft-xtask` is the library `crates/xtask` (package `craft-xtask`), separate from #9's root
      `xtask` binary, and #9's `ci.rs` would call `craft_xtask::runner` once both exist.
    - storytold/craft-libs#698 (whatever-industries, conflicting) changes
      `crates/audio/tests/codecs/testkit.rs`, which the `fixtures` move rewrites.

    Open craft-common-actions pull requests change `gate.yml`, whose lines this RFC cites:
    storytold/craft-common-actions#124 (bmh129, conflicting) adds an optional cache for files the gate
    downloads, and #220 (dylanmtaylor) locks the workflow actions through a lockfile. The `gate.yml`
    line numbers in "What exists" are read again before the `runner` pull request.

    The `test-support` pull requests for `craft-pdf` and `craft-xlsx`, the path-filter change and the
    `fixtures` move of `crates/audio/tests/codecs/testkit.rs` merge with these pull requests or wait for
    them. The `input` module follows whichever egui revision `craft-ui` has when it lands.
13. **Command-line conventions.** Open RFC-0012 (storytold/craft-contrib#18) has each CLI's `main` pass
    its arguments and output streams to a library `run(args, out, err) -> exit code` that tests call
    directly. It also has tests pin `--help`, exit codes 0, 1 and 2, and that a protocol subcommand
    such as MCP writes nothing else to stdout. If RFC-0012 is accepted, `bin` adds a helper that checks
    a child's exit code and streams against those rules, and `mcp` checks that the child writes only
    protocol lines on stdout.

## What this RFC decides

- The family: `craft-testkit` and its modules and features, the `craft_ui::testing` additions, the
  `test-support` rule for format crates, and `craft-xtask` and its modules, with their boundaries.
- The source of each part's code and its first consumers.
- The rules for every crate, including the reading of required variables.
- The move plan and what stays in the apps.
- The migrations onto crates that exist: each app copy in that section's table moves onto its
  craft-libs crate, with the missing cases ported to the crate first.

The new crates under "Candidates for later RFCs" are not decided here.

The affected repositories are craft-libs and every Craft application with a copy listed above:
VectorCraft, PhotoCraft, DesignCraft, LightCraft, FilmCraft, EffectCraft, PdfCraft, GridCraft, WordCraft,
DeckCraft, SoundCraft, CADCraft, FigCraft, AniCraft, MaterialCraft, TextureCraft, craft-launcher and
Craft Anything.

Work on the parts after `guards` starts after this RFC is accepted. Per RFC-0000 this is a 3-day
comment period change.
