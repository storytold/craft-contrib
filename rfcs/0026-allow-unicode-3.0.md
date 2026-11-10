# RFC-0026: Add `Unicode-3.0` to the licence allowlist

Every Rust Craft app already ships code under `Unicode-3.0`, but the
licence allowlist in `standards/house-rules-rust.md` doesn't name it. The
rules say a repository may not add an identifier by editing its own copy,
so every app is out of step with its own rules today, and a reviewer has
to stop and ask about crates that every app already depends on.

## Where it comes from

- `unicode-ident`, which `proc-macro2` and so nearly every Rust build
  depends on, is `(MIT OR Apache-2.0) AND Unicode-3.0`.
- The ICU4X crates (`icu_normalizer`, `icu_properties`, `icu_casemap` and
  others) are `Unicode-3.0`. `url` pulls them in through `idna`. On
  2026-11-08 the `Cargo.lock` on `main` of each of the 17 Rust app
  repositories held 11 to 18 `icu_*` crates. PhotoCraft added
  `icu_casemap` directly in #11671 and #10555.
- PdfCraft's and Craft Anything's `deny.toml` already allow it. The other
  apps' `deny.toml` files don't, which is why the gap hasn't shown up as
  a failing check.

## The licence

The Unicode License v3 is a short permissive licence, OSI-approved. It
allows use, copying, modification and distribution, including in
proprietary products, as long as the copyright and permission notice are
kept with copies of the data or software. That is the same obligation as
`MIT` or `BSD-3-Clause`, and `ATTRIBUTION.md` and `NOTICE` already meet
it.

## Decision

- Add `Unicode-3.0` to the licence allowlist in
  `standards/house-rules-rust.md`.
- Each repository picks up the change the next time it copies the Rust
  block into its `AGENTS.md`. A repository whose `deny.toml` lists allowed
  licences may add `Unicode-3.0` there in the same change.

## Not in this RFC

- Other identifiers that some repositories' `deny.toml` allows but the
  list doesn't (for example `Zlib` and `BSL-1.0` in PdfCraft's), and the
  all-permissive notice on the Norwegian hyphenation patterns that
  DesignCraft bundles. Each needs its own decision.
