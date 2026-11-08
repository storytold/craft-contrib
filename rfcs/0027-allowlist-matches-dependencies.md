# RFC-0027: The licence allowlist covers what the apps depend on

The shared licence allowlist in `standards/house-rules-rust.md` says a
repository may name a shorter list but may not add to it. In practice the
apps depend on more than the list allows, and some repositories have written
their own, broader lists:

- FilmCraft's and EffectCraft's `AGENTS.md` allow "MIT/Apache-2.0/BSD/ISC/
  Zlib/Unicode/CC0/BSL-1.0, or MPL-2.0 used unmodified".
- PdfCraft's `deny.toml` allows `MIT-0`, `Zlib`, `Unicode-3.0`, `BSL-1.0`
  and `Apache-2.0 WITH LLVM-exception`.
- Craft Anything's `deny.toml` also allows `0BSD` and `CDLA-Permissive-2.0`.

So the rule and the repositories disagree, and a reviewer can't tell which
one holds. This RFC settles it by making the shared list match what the apps
actually ship, on the same terms each repository already applies.

## What the apps depend on

On 2026-11-08, every crates.io package in the `Cargo.lock` on `main` of the
17 Rust app repositories (1,598 crate versions) was looked up on crates.io.
These licences are needed and are not on the list, counting only packages
whose licence expression no listed licence satisfies:

| Licence | Apps | Typical crates |
|---|---|---|
| `Unicode-3.0` | 17 | ICU4X `icu_*` (through `url`/`idna`); covered by RFC-0026 |
| `Zlib` | 17 | `foldhash` (through `hashbrown`), `slotmap`, `zlib-rs` |
| `BSL-1.0` | 17 | `clipboard-win`, `error-code` (through `arboard`, Windows clipboard) |
| `MPL-2.0` | 15 | `option-ext` (through `dirs`), `colored`, `symphonia` |
| `CDLA-Permissive-2.0` | 15 | `webpki-roots`, `webpki-root-certs` (TLS root certificates, data) |
| `Apache-2.0 WITH LLVM-exception` | 6 | `cranelift-*`, `wasmtime-*` |
| `IJG` | 3 | `jpeg-encoder` (with MIT OR Apache-2.0) |
| `NCSA` | 2 | `libfuzzer-sys` (fuzz targets only) |
| `0BSD` | 1 | `yaxpeax-arch`, `yaxpeax-arm` |
| `Unlicense` | 1 | `ksni` |
| `MPL-2.0+` | 1 | `imbl` |

The lockfiles include build, test and every-platform dependencies, so not
every crate above ships in every build. `Zlib`, `BSL-1.0` and
`CDLA-Permissive-2.0` ship in all or nearly all of them.

## Decision

1. Add these identifiers to the allowlist: `Zlib`, `BSL-1.0`, `0BSD`,
   `Unlicense`, `IJG`, `NCSA`, `CDLA-Permissive-2.0` and
   `Apache-2.0 WITH LLVM-exception`. Each is permissive: keep the notice,
   and in IJG's case say the software is based in part on the IJG's work.
   `ATTRIBUTION.md` and `NOTICE` already carry those obligations.
2. Allow `MPL-2.0` for third-party dependencies used unmodified, as FilmCraft
   and EffectCraft already do. MPL-2.0 is file-level copyleft: changes to its
   files must be published under MPL-2.0, but the rest of the program isn't
   affected. Vendoring and patching an MPL-2.0 crate, or copying its code into
   our files, still needs a maintainer's approval. `MPL-2.0+` counts as
   `MPL-2.0`.
3. Once this merges, local lists may only narrow the shared list, as the rule
   already says. FilmCraft's and EffectCraft's own lines, and any `deny.toml`
   allow list, are reduced to the shared list or a subset of it, in each
   repository's next change to those files.

`Unicode-3.0` is decided by RFC-0026, not here. `MIT-0`, in PdfCraft's
`deny.toml` but not in use, is left out: an identifier is added when
something depends on it.

## Not in this RFC

- A shared `cargo deny` check in `craft-common-actions`, so the list is
  enforced rather than only written down. Only PdfCraft and Craft Anything
  have a `deny.toml` today. That is a follow-up for `craft-common-actions`.
- Asset licences, such as fonts and hyphenation patterns, which follow the
  same list but weren't scanned here.
