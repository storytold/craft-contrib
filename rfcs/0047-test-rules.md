# RFC-0047: Test rules

## Summary

Adopt `standards/testing.md` as the rules for every test in the Craft repositories, written by a person or an agent. The document is a living document under [RFC-0013](0013-amending-rfcs.md) point 5. It says what a test is for, which tests to write and which to cut, how to judge an existing test, and where tests go. The shared agent instructions carry a short form that links to it.

## Motivation

We have more tests than they are worth. One Windows run of craft-libs' workspace runs 13,660 tests, and 76% of them finish in under 0.1 s, so their cost is mostly compiling and linking on three operating systems. craft-libs `main` has 304 integration test binaries in 78 crates; 43 crates have more than one. One binary per crate would remove 226 links on each operating system in every CI run. Many tests check things a reader can verify, or repeat what another test already checks.

The rules are built so that two people judging the same test reach the same verdict. Each test is cut, trimmed, rewritten, or kept with its invariant and the change it catches stated. Each category is one question about the test.

The rules were applied to every test in three craft-libs crates ([craft-libs#2146](https://github.com/storytold/craft-libs/pull/2146)):

| Crate | Tests run | Test time | Cut | Trimmed | Rewritten |
|---|---|---|---|---|---|
| craft-xlsx | 101 → 92 | 22.3 s → 0.9 s | 9 | 11 | 4 |
| craft-menu | 70 → 59 | 1.1 s → 0.7 s | 13 | 12 | 3 |
| craft-text | 69 → 50 | 2.5 s → 0.2 s | 26 | 15 | 4 |

97 of 255 tests were cut, trimmed or rewritten, and the diff removed about 930 lines. Most of the time saving came from rewrites: four craft-xlsx limit tests built millions of records. Most of the removed lines came from cuts.

## Proposal

1. Adopt `standards/testing.md`, added by this RFC's PR, as the rules for every test.
2. It is a living document under RFC-0013 point 5. Wording fixes are ordinary PRs. Adding, removing or changing a rule needs an RFC that updates this one.
3. Three rules use MUST or MUST NOT:
   - A test is never deleted, weakened, ignored or skipped to make a check pass. A quarantine of a flaky test, as `testing.md` describes, is the only exception.
   - A pull request that cuts or quarantines tests lists every removed test. Such a pull request SHOULD contain no other change.
   - A test that fails every time is never quarantined.

   Everything else uses SHOULD, SHOULD NOT or MAY, as [RFC-0048](https://github.com/storytold/craft-contrib/pull/72) point 2 sets for the shared rules.
4. The shared agent instructions carry a short form of the rules and link to the full document, so agents' context stays small. The short form is rules 27 to 32 of the shared rules file, `standards/app-agents.md` ([RFC-0048](https://github.com/storytold/craft-contrib/pull/72)). Where the two differ, the shared file wins, and this document is corrected to match.
5. RFC-0048 merges only after this RFC is accepted, and the two merge back to back: `testing.md` links `clean-room.md`, which RFC-0048 adds, and RFC-0048's rules 27 to 32 link `testing.md`.
6. Existing tests are judged and cut only when an issue asks for it, per crate, in a pull request that lists each removed test with its category.

## Alternatives

- **Short rules in the agent instructions only.** Agents get the rules in their context, but a short form can't hold the category questions and examples that make two judges agree.
- **Cut tests by a coverage or mutation score.** A score finds gaps but doesn't say which tests repeat another or check a reader-verifiable line. Coverage stays a check that points at gaps.
- **No shared rules; each repository decides.** Verdicts then differ between repositories and between reviewers, and each cut is argued again.
