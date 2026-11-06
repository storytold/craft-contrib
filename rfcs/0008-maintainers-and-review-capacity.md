# RFC-0008: Roles and review capacity

Related: storytold/photocraft#5439.

## Summary

Write down the contributor roles and what each may do, and name review as
the bottleneck to plan around. Maintainers stay the ArtCraft team: owners
decide who on the team reviews and merges which app. This RFC adds no path
for people outside the team to become maintainers.

## Motivation

Review is now the bottleneck. Across the storytold organization, PRs opened
and merged per day (from GitHub search, as of 2026-10-15):

| Day | Opened | Merged |
|---|---|---|
| Oct 5 | 316 | 290 |
| Oct 8 | 875 | 626 |
| Oct 11 | 2058 | 1034 |
| Oct 13 | 2214 | 1205 |
| Oct 15 | 1132 | 692 |

Since Oct 10, about 1000 more PRs are opened than merged every day. About
4500 PRs are open, and almost all of them were opened in the last 7 days.

The wait for a merge has grown too. Median time from opening a PR to merging
it, for PRs merged that day:

| Merged on | Median wait |
|---|---|
| Oct 5–10 | under 3 hours |
| Oct 11–13 | about 9 hours |
| Oct 14–15 | about 21 hours |

This counts only PRs that were merged, so PRs still waiting make the real
wait longer.

Share of PRs already handled (merged or closed), by the day they were opened,
as of Oct 15:

| Opened on | Opened | Handled |
|---|---|---|
| Oct 5 | 316 | 99% |
| Oct 8 | 875 | 95% |
| Oct 10 | 2008 | 88% |
| Oct 11 | 2058 | 79% |
| Oct 12 | 2115 | 75% |
| Oct 13 | 2214 | 54% |
| Oct 14 | 1661 | 37% |

The PRs themselves did not get bigger or harder to review. The median PR
changed 98 to 279 lines in 3 to 7 files on every day from Oct 4 to Oct 15,
with no upward trend (sample of 300 PRs per day). The wait grew because
there are more PRs, not because each one takes longer to check.
The team simply has far more PRs than time to review them.

## Proposal

1. **Roles**
   - **Contributor**: anyone who opens issues or PRs. On Discord, contributor
     tiers follow merged PRs, after linking a GitHub account:
     - **Junior Contributor**: 1 merged contribution;
     - **Contributor**: 10 merged contributions, or a large feature;
     - **Senior Contributor**: given by hand to professionals with a large
       contribution.
     Tiers are given by these rules, not on request.
   - **Maintainer**: a member of the ArtCraft team in `craft-app-team`, as in
     RFC-0000. Owners decide who is on it.
   - **Owner**: as in RFC-0000.
2. **More review from the team.** Owners can give team members review and
   merge rights for specific apps, so each app has someone responsible for
   its queue.
3. **Less review work per PR.** The other RFCs exist to cut the work a
   reviewer does per PR: fewer duplicates (RFC-0003, RFC-0004), PRs that are
   quick to check (RFC-0005), and agents that review their own diff before
   opening a PR and answer review comments before opening more (RFC-0006).

## Alternatives

- **Change nothing.** The numbers above show the queue growing every day.
- **Give merge rights to active outside contributors.** Not proposed: merge
  rights stay with the team.
