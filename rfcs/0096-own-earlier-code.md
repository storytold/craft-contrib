# RFC-0096: A contributor's own earlier code

Updates: [RFC-0072](0072-shared-agents-md.md)

## Summary

A contributor MAY port code they wrote, and hold the right to license, from their own project outside storytold, when that code was itself made within the clean-room rules. The pull request says so in one line, naming the project and its licence. For [clean-room.md](../standards/clean-room.md) §2 and rule 23 of the shared `AGENTS.md`, such code is not "another product's code".

## Motivation

Rule 23 and clean-room.md §2 say a contributor MUST NOT copy another product's code. §5 says a format SHOULD NOT be implemented from another implementation's source outside storytold. None of them separates someone else's implementation from the contributor's own, so a maintainer can fairly block a port of the contributor's own code under the MUST NOT.

A contributor who has already built, in their own project, what an app needs therefore has two choices:

- **Rewrite it from scratch.** The new code is slower to arrive and usually worse, because the earlier code's tests, its fixes for malformed input and its edge cases are thrown away.
- **Port it and argue provenance on every pull request.** The review then turns on a rule whose purpose isn't at stake.

The rules exist to keep copyleft and proprietary knowledge out and to keep the code ours to license. Both purposes are met when three things hold: the contributor owns the code, can license it to the repository, and made it under the same rules.

## Proposal

1. **§2 says what "another product" excludes.** "Another product's code" does not include code you wrote yourself and hold the right to license, when it meets point 2. Rule 23's wording does not change. Its meaning is set by clean-room.md, which it already links. Every `AGENTS.md` copy therefore keeps its hash.
2. **Such code was itself made within these rules:**
   - from the sources §1 lists;
   - no GPL, LGPL or AGPL code read; observing a copyleft tool only as §4 allows here;
   - no proprietary internals (§3);
   - nothing copied from another product (§2).

   Anything in the earlier project that does not meet this MUST NOT be ported.
3. **Ownership is the contributor's to get right.** It qualifies only if you hold the right to license it, not your employer or a client. Code with other authors qualifies only when every author agrees. Code written with a coding agent counts as yours on the same terms as anything you contribute here. Code you only use or maintain does not count.
4. **The pull request says so in one line**, for example: "Adapted from my own project X (MIT)." No itemised list, hashes or SHOULD departure note is needed.

   §1 still says a statement is not evidence. A reviewer who finds code shaped like an implementation other than the named project may ask about it, as today.
5. **§5 gains the same exception:** "…outside storytold, whatever its licence, other than your own code as §2 allows."
6. **The ported code meets every repository rule as new code would:** tests, never-panic, layering, and the licence allowlist for anything it depends on.

This RFC changes `standards/clean-room.md` §2 and §5. It adds `Updated by: RFC-0096` to RFC-0072, which created clean-room.md as RFC-0048. Comment period: 3 days, per RFC-0013, unless a maintainer sets longer.

## Alternatives

- **Keep the rule and treat each port case by case.** Every reviewer decides alone, so a contributor can't know in advance whether to port or to rewrite.
- **Allow it with no note at all.** It is cheaper, but it removes the one line that lets a reviewer check the claim against the named project.
- **Add a licence test** ("under a licence on licences.md"). This is too wide: the allowlist includes share-alike and asset licences, and licences that cover unmodified dependencies only. Ownership is the real condition, because the author can license their own code to the repository whatever licence the earlier project used.

## Affects

The whole org: every Craft repository, through `standards/clean-room.md`. Rule 23's text is unchanged.
