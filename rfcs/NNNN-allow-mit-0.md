# RFC-NNNN: Add `MIT-0` to the licence allowlist

RFC-0027 left `MIT-0` out because nothing depended on it: "an identifier is added when something depends on it".
Something now does. `constant_time_eq`, licensed `CC0-1.0 OR MIT-0 OR Apache-2.0`, is a dependency of the official
`blake3` crate and of `zip`, and so of craft-libs' `craft-zip`. An app that enforces the list as written
(every identifier in an expression must be on it) can't use any of the three.

Affects: every Rust Craft app and shared library (the allowlist applies to all of them).

## Where it comes from

- `constant_time_eq` (about 400 million crates.io downloads, 170+ dependent crates) is a small crate for comparing
  byte strings in constant time.
- `blake3` depends on it unconditionally. BLAKE3 is the hash several Craft designs name for content addressing and
  file integrity.
- `zip` depends on it, so craft-libs' `craft-zip` brings it into every app that reads or writes ZIP-based files.
- FantasyMapCraft hit both on 2026-11-21. It wrote its own BLAKE3 (adapted from the public-domain reference
  implementation) and its own stored-only ZIP writer rather than break the list. Those are workarounds the suite
  shouldn't need.

## Why the usual reading doesn't save it

The allowlist says an expression is allowed "when every identifier in it is on this list". `CC0-1.0` and
`Apache-2.0` are on it, but `MIT-0` is not, so the expression fails, although a user could take the crate under
either listed licence. Changing that reading to "at least one alternative of an `OR`" is a broader change, and some
repositories enforce the strict reading on purpose. This RFC keeps the strict reading and adds the one identifier.

## The licence

MIT No Attribution is the MIT licence without its condition that the copyright and permission notice be kept in
copies. It is OSI-approved and strictly more permissive than `MIT`, which is on the list. It places no obligation on
us. Apps still record the crate in `ATTRIBUTION.md` and `NOTICE` as they do for every dependency.

## Decision

- Add `MIT-0` to the allowlist in `standards/licences.md`.
- A repository whose `deny.toml` lists allowed licences may add `MIT-0` in the same change.

## Not in this RFC

- Reading `OR` expressions as "one listed alternative is enough". That would also admit this crate, but it changes
  how every expression is checked, so it needs its own decision.
