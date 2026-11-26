# RFC-NNNN: Hyphenation is read through craft-proof

Updates: [RFC-0011](0011-language-data-in-craft-i18n.md)

## Summary

RFC-0011 says the code that reads Language data stays in craft-libs:
"craft-proof for spelling, craft-text for hyphenation". This RFC changes
that to craft-proof for both. craft-text keeps line breaking, bidi and
segmentation, and no longer finds hyphenation points itself.

## Motivation

craft-proof already reads hyphenation patterns from craft-i18n's
`language-data/`, with English embedded and other languages from Language
packs. WordCraft and VectorCraft hyphenate through it. craft-text's
hyphenation was a second copy, with a bundled Moby list and patterns plus
`hypher` language features. A survey of every app's `main` and of
craft-libs on 2026-11-26 found no caller of it, and no app enabling one of
its language features.

Pointing that copy at craft-i18n, as RFC-0011's "Open" list expected, would
keep two engines reading the same data, and two apps could break the same
word differently. Removing it leaves one engine.

## Proposal

1. **craft-proof reads hyphenation patterns**, as it reads spelling
   dictionaries. Apps hyphenate through `craft_proof::hyphen`.
2. **craft-text has no hyphenation of its own.** Its line breaker takes
   hyphenation points from the caller as before. Its bundled hyphenation
   data, its `hypher` dependency and its per-language hyphenation features
   are removed (craft-libs#3737).
3. In RFC-0011's "What this asks of each app", the first point reads as
   "Read spelling and hyphenation through craft-proof."
4. RFC-0011's open item "craft-text still bundles its own English patterns
   and `hypher` features" is closed by point 2.

Languages that only `hypher` covered (as, bn, fi, gu, kn, ku, la, ml, mr,
or, pa, sa, te, tk) have no craft-i18n patterns yet. They are added to
craft-i18n's manifest under its licence rules when an app needs them.

DesignCraft and VectorCraft still carry local hyphenation code. Under
RFC-0011, each moves onto craft-proof in its own repository.

This RFC updates RFC-0011, so its PR adds `Updated by: RFC-NNNN` to
RFC-0011.

## Alternatives

- **craft-text reads craft-i18n's patterns itself.** It would need its own
  loader and pattern engine beside craft-proof's, for code nothing calls.
- **craft-text forwards to craft-proof.** It would add a dependency and an
  API with no callers. An app that needs hyphenation can depend on
  craft-proof directly.
