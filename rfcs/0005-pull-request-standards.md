# RFC-0005: What a pull request should look like

## Summary

One shared checklist for PRs in every craft app: a narrow fix, a test that
fails without it, a description a reviewer can check in a minute, drafts only
while work is unfinished, and quick answers to review comments. Repository
`AGENTS.md` files keep their own stricter rules on top.

## Motivation

Review is the slowest step in the whole process (see RFC-0008). Every PR that
is hard to read, too wide, or left as a draft after it is ready costs review
time. The repositories already have good rules in their `AGENTS.md` files, but
they differ, and a lot of what makes a PR easy to merge is not written down
anywhere.

## Proposal

1. **Scope.** Fix only what the issue describes, where the bug is. No drive-by
   refactors or formatting changes in unrelated code. If you notice the same
   bug on another path, fix it in the same PR; if you notice a different bug,
   open an issue.
2. **Test.** Add a test that fails without the fix and passes with it. Prefer
   real data from the repository's test files over mocks when they exist.
3. **Description.** In this order, short:
   - `Fixes #N.` only when the PR fixes the whole issue; otherwise
     `Part of #N.` and what is left.
   - What the user saw.
   - The cause.
   - What changed, and why the change is this wide and no wider.
   - The test: its name and what it checks.
4. **Title and commits** follow the repository's existing style (check the
   last few commits on `main`).
5. **User-facing text** goes through the repository's translation system, with
   a line in every catalog the repository requires.
6. **Drafts.** Open as a draft only while the work is unfinished. Mark it ready
   as soon as it is done and the project's checks pass. A finished PR left as a
   draft is invisible to reviewers.
7. **Red `main`.** If a check fails on a test your PR does not touch, check
   whether the same test fails on `main`. If it does, say so on the PR with a
   link to the failing run on `main`, instead of changing unrelated code.
8. **Answer reviews quickly.** Reply to review comments within a day, or say
   when you will. A PR with an unanswered question for more than 3 days may be
   closed by a maintainer and reopened later.
9. **Keep it current.** If `main` moved and the PR conflicts, rebase it and
   check that the fix is still needed (the bug may have been fixed meanwhile).

## Alternatives

- **Leave it to each repository's `AGENTS.md`.** Those stay and can be
  stricter, but most of this is the same everywhere and is easier to find in
  one place.
- **A PR template only.** A template is the way to apply this RFC, not a
  replacement for agreeing on the rules.

## Open questions

- Is 3 days the right limit for an unanswered review question? Agents answer in minutes, but the person running them may be away over a weekend.
