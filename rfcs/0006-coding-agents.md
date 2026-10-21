# RFC-0006: A shared section for coding agents (`AGENTS.md`)

Related: storytold/craft-libs#37, storytold/craft-libs#38.

## Summary

Add one shared section to every craft app's `AGENTS.md` with the rules that
apply to all of them: claim before working, verify instead of guessing, keep
fixes narrow, review before opening a PR, and don't flood the repositories.
Each repository keeps its own rules below it.

## Motivation

The craft apps welcome AI-assisted contributions, and much of the current PR
volume comes from coding agents. An agent follows its instructions literally,
so gaps in the instructions turn into the same mistake repeated many times:
duplicate PRs, fixes for bugs already fixed on `main`, PRs left as drafts,
or code that uses functions that don't exist.

Running agents on hundreds of issues across eleven repositories taught us
which instructions prevent those mistakes. This RFC proposes them as the
shared part.

## Proposal

The shared section says:

1. **Claim and check** as in RFC-0003: right before each issue, read its
   comments, search PRs for `#N` and keywords, and check recent commits on
   `main`. Claim on the issue before writing code. Release claims you won't
   finish.
2. **Verify, don't guess.** Before using a function, type or field, open its
   definition and use it exactly as defined. If you cannot build or run the
   tests locally, say so to the human running you, and rely on the project's
   CI before marking the PR ready.
3. **Notes from triage can be wrong.** A short note on where the bug is (from a
   person, a bot, or another agent) is a starting point. Confirm the cause in
   the code before fixing it.
4. **Narrow fixes and real tests**, as in RFC-0005.
5. **A second pass before opening the PR.** Review the diff against the issue:
   every identifier exists as used, the test fails without the fix, nothing
   outside the bug changed, and the description is true.
6. **Look for a better PR** before opening yours, as in RFC-0004, and close
   yours when the other is better.
7. **Don't flood.** Open PRs at a pace the maintainers can review, and don't
   open new ones while earlier ones have unanswered review comments.
8. **Be gentle with GitHub.** Reuse search results, avoid polling loops, and
   stop when you hit a rate limit instead of retrying in a loop.
9. **A human is responsible.** The person running the agent reads review
   comments and answers them, and is accountable for what the agent opens.

## Alternatives

- **Separate rules per repository.** This is the current state; the same
  lessons have to be learned again in every repository.
- **Ban or limit AI-assisted PRs.** The craft apps chose to welcome them; this
  RFC keeps that choice and makes the volume easier to handle.

## Open questions

- Should agents mention in the PR that they were used? Each repository's
  current `AGENTS.md` decides this today.
