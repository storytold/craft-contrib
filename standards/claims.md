# Claims and duplicate pull requests

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as in RFC 2119 and RFC 8174.

These rules apply to every issue and pull request in the Craft repositories, for people and agents. `AGENTS.md` carries the short form and links here. Changing a rule needs an RFC.

## Before you start

1. You SHOULD read the issue's comments and search open and closed pull requests for `#N` in the same repository (`storytold/other-repo#N` is a different issue).
2. You SHOULD check recent commits on `main` that touch the area, and search open pull requests and issues by keywords. If the issue is already fixed or covered, say so on the issue with a link and SHOULD NOT fix it again.
3. You SHOULD claim with a comment on the issue, for example `Working on this.` These also count as claims: an open pull request that says `Fixes #N` or `Part of #N`, and a maintainer's comment that they are fixing it. Bot comments, reactions, assignments without a comment and messages outside GitHub are not claims.
4. A claim is live while either holds:
   - its pull request is open;
   - its comment is under 24 hours old, or a visible sign of progress followed it: a draft pull request, a commit pushed to a linked pull request, or an update comment.
5. Taking over:
   - A comment-only claim with no progress after 24 hours has expired. You MAY take the issue over by saying so on it.
   - For a claim with an open pull request, you SHOULD ask its author on the issue or the pull request first, and SHOULD take over only if they do not answer within 24 hours, saying so on the issue.
   - You SHOULD credit their work if you build on it (`Co-authored-by`, and keep the authorship of their commits).
6. If you stop, get stuck or find a better pull request, you SHOULD say so on the issue with what you found.
7. On an umbrella issue, you SHOULD claim only the part you work on, or its sub-issue.
8. Maintainers MAY assign the issue to the claimant. Agents working through a backlog SHOULD run steps 1 and 2 right before each issue, not once for the whole list.

## When two pull requests fix the same thing

1. Right before opening a pull request, you SHOULD search open pull requests for the issue number and for the functions and files you changed, also in a repository the code is moving to.
2. Pull requests for independent parts of an issue are not duplicates. Compare the behaviour each one changes. A pull request that waits for another SHOULD link it in its description.
3. You SHOULD compare duplicates with these questions:
   - Does it fix only where the bug is?
   - Does it handle the case the way neighbouring code handles it?
   - Does it cover every point of the issue and every way a user reaches the bug, with a test for each?
   - Does it update the docs and help that describe the behaviour?
4. If theirs is better on at least one question and worse on none, you SHOULD close yours with a comment that links theirs and says why. If yours is better that way, you SHOULD say what differs in one factual sentence on both pull requests. If neither is, it is a tie: the earlier valid claim wins ("Live claims" point 2), otherwise a maintainer decides (point 6). Who opened first does not matter, except under "Live claims".
5. If each has something the other lacks, one author SHOULD add the missing part to the other pull request with `Co-authored-by`, and the other SHOULD close theirs. If one pull request adds a general mechanism the other could use, the specific one SHOULD wait and be rebuilt on it.
6. When the authors disagree, a maintainer decides with the same questions and explains on both pull requests.

## Live claims

1. You SHOULD NOT open a pull request on an issue with a live claim.
2. If two fixes are equal, the earlier valid claim wins.
3. A pull request that ignored a live claim SHOULD replace the claimed work only if it is better on at least one question in "When two pull requests fix the same thing" and worse on none, and says why. Otherwise a maintainer MAY close it unreviewed.
4. Maintainer step: an author who ignores live claims more than once SHOULD get the `ignored claim` label on the pull request and a warning that counts in every Craft repository for 90 days. While a warning counts, that person's pull requests on claimed issues are closed without review.
