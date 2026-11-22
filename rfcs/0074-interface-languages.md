# RFC-0074: A build chooses the interface languages it embeds

Related: RFC-0011 (storytold/craft-contrib#15, language data, `CRAFT_PROOF_EMBED`), RFC-0056
(storytold/craft-contrib#56, which leaves the interface catalogs to "its own decision, in its own
RFC"), RFC-0023 (web builds' memory budgets), RFC-0060 (storytold/craft-contrib#60, mobile
builds), RFC-0072 (the shared app rules), storytold/vectorcraft#3432 (closed).

## Summary

Today a build embeds every interface language, or, in an app on craft-i18n, none. An app on
craft-i18n embeds the catalog of every locale that has one when `CRAFT_I18N_DIR` is set, and is
English only without it; its integration guide adds that "Web (wasm) builds embed the same
locales as desktop". An app that keeps its own catalogs, as VectorCraft does, embeds all of them
with no switch at all.

This RFC adds one build variable, `CRAFT_I18N_EMBED`, naming the interface locales a build embeds,
in the same form as RFC-0011's `CRAFT_PROOF_EMBED` (`de,fr`, `all` or `en`). Unset, it is `all`,
so every existing build is unchanged. A web build, or any build with a size budget, can then ship
the languages its own users need and no others.

## Motivation

- **The catalogs are a large share of a web build, today.** PhotoCraft is on craft-i18n and has a
  web build, whose CI job ("web build and wasm size gate") builds with `CRAFT_I18N_REQUIRED=1`
  and so embeds every locale that has a `photocraft.tsv`. At its pinned craft-i18n commit
  (e56d262) that is 35 locales: 6,438,689 bytes of `photocraft.tsv` and 850,328 of `common.tsv`,
  7,289,017 in all (2,241,533 with `gzip -9`, 1,254,307 with `brotli -q 11`). (Measured by
  @hectorviov on this pull request.)
- **And growing.** On 19 November (bedbbad72f) VectorCraft's 28
  interface catalogs are 20,885,397 bytes as embedded: 6,073,327 bytes with `gzip -9` and
  3,053,717 with `brotli -q 11`. On 22 October (cee30f05b) the same directory held 24 catalogs
  and 7,060,092 bytes, so they nearly tripled in four weeks as languages and strings were added.
  A desktop download can carry that. A web build's WebAssembly
  is downloaded and compiled on every first visit, and RFC-0023 asks web builds to live within
  stated budgets.
- **The choice today is all or nothing, or just all.** In an app on craft-i18n, leaving
  `CRAFT_I18N_DIR` unset gives English only and setting it gives every language. VectorCraft,
  which embeds its own catalogs with `include_str!`, always gives every language. A page that
  serves German and French users has no build between the two.
- **The language list at run time does not help.** The app's offered list (`I18n::new`'s
  `offered`) narrows which languages the picker shows, but every embedded catalog is still in the
  binary.
- **It was raised as a VectorCraft pull request, and held for this kind of decision.**
  storytold/vectorcraft#3432 proposed one Cargo feature per interface language, all on by default.
  A maintainer held it because it "changes how the app is built and packaged, which is the owner's
  call rather than a behaviour fix". This RFC is that decision, made once for the suite, in the
  form craft-i18n already uses for its build input.

## Proposal

### 1. The variable

1. `CRAFT_I18N_EMBED` names the interface locales a build embeds: a comma-separated list of
   craft-i18n locale codes (`de,fr,pt-br`), `all`, or `en`. Unset or empty means `all`.
2. It is read where `CRAFT_I18N_DIR` is read: the app's `build.rs`, which writes the `CRAFT_I18N`
   static. A locale it doesn't name is left out of the static, so its catalogs (the shared one and the app's) are not in the binary.
3. `en` embeds no catalog: English is the source text.
4. A named locale brings its fallback chain, as the `craft-i18n` crate builds it: a locale with a
   region subtag falls back to the locale without it, when that one has a catalog (`pt-br` to
   `pt`; a script subtag never falls back, so `sr-latn` does not bring `sr`). No locale on
   craft-i18n `main` has such a parent today (it has `pt-br` and no `pt`), so this rule applies
   only once one is added.
5. A named locale with no catalog in the checkout is a `cargo::warning`, or a `cargo::error` under
   `CRAFT_I18N_REQUIRED=1`, as a missing checkout is today.
6. `cargo::rerun-if-env-changed=CRAFT_I18N_EMBED`.

### 2. What the app shows

1. A locale that is not embedded is not offered, even when the app's own list offers it. The
   picker, `from_code`, `match_tag` and `system_lang` behave as if the app didn't offer it.
2. A system locale that isn't embedded falls through to the next locale in the system's list that
   is, and then to English.
3. A saved interface-language setting naming a locale that isn't embedded reads as "Automatic":
   the app follows the system locale, and its language menu and preferences show Automatic. The
   saved value is kept, so a build that embeds it again shows it again. Setting such a locale (by
   the app's command) is accepted and kept the same way; a code no catalog exists for is still
   refused.

### 3. Apps not yet on craft-i18n

An app that still embeds its own catalogs honors the same variable in its own build, with the
same meaning, so a packager learns one switch. VectorCraft is such an app: its catalogs are
the evidence of growth above, and it would read the variable in `vectorcraft-ui-egui`'s build. (A fork of
VectorCraft, addibble/vectorcraft, does this today with one Cargo feature per language and
section 2's behavior, without section 1.4's fallback chain: its `pt-br` catalog is complete on
its own. That form is non-normative, and the variable replaces it.) The shared app rules
(RFC-0072, standards/app-agents.md point 16) say translations are added in storytold/craft-i18n. When
such an app moves its catalogs there, the variable keeps its meaning.

### 4. Web builds

The integration guide's "Web (wasm) builds embed the same locales as desktop" becomes "Web (wasm)
builds embed the locales `CRAFT_I18N_EMBED` names, `all` by default".

## What this leaves alone

- Language data (spelling, hyphenation, word lists): RFC-0011 and `CRAFT_PROOF_EMBED`.
- Which languages a release ships. Releases keep `all` unless their maintainers choose otherwise.
- Fetching catalogs at run time. A web build that downloads a language when the user picks it
  would be a later RFC, through the same consent and checks as Language packs.

## Acceptance tests

1. `CRAFT_I18N_EMBED=en` gives a binary with no catalog. The picker lists English only.
2. `CRAFT_I18N_EMBED=de` offers English and German only. A saved `fr` setting, or `fr` set by the
   app's command, reads as Automatic and is kept; an unknown code is refused.
3. With a test checkout that has `pt` and `pt-br`, `CRAFT_I18N_EMBED=pt-br` embeds `pt` too, and a
   string `pt-br` lacks falls back to `pt`.
4. A system locale list of `ja, de` with only `de` embedded shows German.
5. Unset, the static is the same as today's.

## Open questions

1. **One variable or two.** A "Language build" might want the same locales for the interface and
   for proofing. Should `CRAFT_I18N_EMBED` also set `CRAFT_PROOF_EMBED` when that is unset?
2. **Web and mobile defaults.** Should a web release, or a mobile one under RFC-0060, default to
   a smaller set, or keep `all` until catalogs can be fetched on demand?

## References

- craft-i18n, the build option `CRAFT_I18N_DIR`:
  https://github.com/storytold/craft-i18n/blob/main/docs/integration.md
- craft-i18n ADR 0002 (lookup in craft-libs, data in craft-i18n):
  https://github.com/storytold/craft-i18n/blob/main/docs/adr/0002-runtime-in-craft-libs-data-in-craft-i18n.md
- The `craft-i18n` crate's offered locales:
  https://github.com/storytold/craft-libs/tree/main/crates/i18n
