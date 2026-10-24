# RFC-0003: Claiming issues

## Summary

Before working on an issue, check that it is still free and still broken,
then say on the issue that you are taking it. A claim expires if nothing
visible happens, and is released when you stop. People and coding agents
follow the same rules, in every craft-app repository.

## Motivation

Several people often fix the same issue at the same time, and find out only
after the work is done. storytold/photocraft#3561 counted **172 issues** with
fixing PRs from two or more authors, and **148** of those PRs closed unmerged
(as of its last update). storytold/pdfcraft#1176 describes the same problem in
PDFCraft. Every duplicate costs the second author their time, and costs a
maintainer a comparison between two PRs that should never have both existed.

A second kind of wasted work is fixing something that is already fixed. Many
reports come from an older release, while `main` already has the fix, or an
open PR already covers it without naming the issue.

Most contributors already comment when they start. What is missing is a
written rule that newcomers can find, plus answers to the cases that keep
causing collisions.

## Proposal

1. **Check that it is free.** Before starting, read the issue's comments and
   search open and closed PRs for its number (`#N`). Only `#N` in the same
   repository counts: `storytold/other-repo#N` is a different issue. If someone
   holds a live claim, pick another issue or ask them on the issue whether they
   still need it.
2. **Check that it is still broken.** Look at recent commits on `main` that
   touch the area, and search open PRs and issues by keywords, not only by
   number. If it is already fixed or covered, say so on the issue with a link
   instead of fixing it again.
3. **Claim with a comment.** Comment on the issue that you are working on it,
   for example `Working on this.` An open draft PR that says `Fixes #N` also
   counts as a claim. A maintainer's comment that they are fixing it counts too.
4. **What is not a claim.** Automated comments (for example "Possibly related"
   from bots), reactions, and messages on Discord or elsewhere. If it is not on
   the issue or in a linked PR, it does not count.
5. **A claim lasts 24 hours** without a visible sign of progress: a draft PR, a
   commit pushed to a linked PR, or an update comment on the issue. After that
   anyone can take the issue, by commenting that they are taking it over.
6. **Release when you stop.** If you give up, get stuck, or find that someone
   else's PR is better, say so on the issue, and add what you found so the
   next person starts further ahead.
7. **One claim per fix.** For an umbrella issue with several parts, claim the
   part you work on (or its own sub-issue), not the whole umbrella.
8. **Maintainers may assign** the issue to the person who claimed it, so the
   claim shows in the issue list and `no:assignee` lists free issues. This is
   optional and needs no tooling.
9. **Coding agents follow the same rules.** An agent working on a backlog does
   steps 1 and 2 right before starting each issue, claims before writing code,
   and releases claims it will not finish.

What happens when two PRs exist anyway is covered by RFC-0004.

The rules go into each repository's contributing guide and PR template, and
into the shared `AGENTS.md` section (RFC-0006).

## Alternatives

- **No written rule, keep commenting informally** (suggested in
  storytold/pdfcraft#1176). This is the current state, and the numbers above
  show it is not enough at this many contributors.
- **Only assignment, no comments.** Most contributors cannot assign themselves,
  so every claim would wait for a maintainer.
- **A bot that assigns on `/take` and drops stale assignments.** Useful later,
  and compatible with this RFC. It can be proposed separately once the rule
  itself is agreed.

## Open questions

- Is 24 hours the right length? (storytold/photocraft#3561 also suggests 24.) Large features may need a longer claim, stated
  in the claiming comment.
- How should this fit the labels and triage process proposed in
  storytold/craft-libs#119 (section 6 there has its own claim rules)?
