# Pull requests

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as in RFC 2119 and RFC 8174.

These rules apply to every pull request in the Craft repositories, by a person or an agent. `AGENTS.md` carries the short form and links here. Changing a rule needs an RFC.

## Shape

1. A pull request SHOULD fix only what the issue describes, where the bug is, and SHOULD fix the same bug on other paths in the same pull request. For a different bug, open an issue unless one exists. It SHOULD NOT change code the fix does not need: no refactors, no formatting.
2. Moving code (to craft-libs, or switching an app onto a shared crate) and changing behaviour SHOULD be separate pull requests.
   - A move SHOULD keep behaviour. Its description SHOULD list the tests that pass and that skip before and after, with counts.
   - A behaviour change SHOULD add a test that fails without it, unless testing.md's C1 exemption applies, and SHOULD list the tests whose result changes.
3. Title and commits SHOULD follow the style of the last commits on `main`.
4. A pull request that waits for another SHOULD link it in its description.
5. A pull request that needs newer craft-libs code SHOULD NOT move the craft-libs `rev` itself. A pull request of its own moves every craft-libs crate to the new `rev`, and the feature pull request waits for it.

## Description

The description SHOULD be short, in this order:

1. First line: what the user gains, for example "Opening a large file is 3x faster".
2. `Fixes #N.` only when the pull request fixes the whole issue; otherwise `Part of #N.` and what is left.
3. What the user saw.
4. The cause.
5. What changed, and why the change is this wide and no wider.
6. The test: its name and what it checks.
7. Verification: what ran and where (CI, a test run, a check in the app), and what did not run. A check that did not run MUST NOT be reported as passed.

Reviewers check the description against this list, and a maintainer MAY ask for a missing section; no CI check enforces it.

A pull request that changes UI SHOULD add the Accessibility section that [accessibility.md](accessibility.md) describes. Screenshots of our own app SHOULD go in the description and SHOULD NOT be committed to the repository.

## Before opening

You SHOULD review the diff against the issue:

- every identifier exists as used;
- the test (if one is required) fails with the fix reverted (run it once that way);
- nothing outside the bug changed;
- the description is true.

## Drafts

You SHOULD open a pull request as a draft only while the work is unfinished, and SHOULD mark it ready as soon as it is done and the checks pass. Reviewers do not look at drafts.

## After opening

1. **Review responses.** You SHOULD respond within 3 days to an actionable review request. A reminder comes at 7 days, and a maintainer MAY close the pull request at 14.
   - The clock starts when an actionable request waits on the author. Optional suggestions, unconfirmed bot findings and a merge conflict do not start it.
   - A fix, an explanation or an agreed plan resets it. Unrelated activity does not.
   - Waiting on a reviewer or a maintainer pauses it and does not count against the author.
   - Closure is a maintainer's decision and is not automatic.
2. **Keeping current.** When a reviewer or maintainer asks, you SHOULD bring the pull request up to date with `main` and check that the fix is still needed. A conflict alone needs no push while the pull request waits for review. On a conflict in a generated file, you SHOULD merge `main` and regenerate the file, and SHOULD NOT resolve it by hand. You SHOULD NOT push to a pull request a maintainer has taken into an integration batch or asked you not to push to.
3. **Red `main`.** If a check fails on a test your pull request does not touch, run or look up the same test on `main`. If it fails there too, you SHOULD link the failing `main` run on the pull request and change nothing unrelated. If it passes on `main`, your change caused the failure, and you SHOULD fix it.
