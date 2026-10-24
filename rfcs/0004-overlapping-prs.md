# RFC-0004: When two PRs fix the same thing

## Summary

When two open PRs fix the same issue, the authors compare them first, and the
better fix wins, not the faster one. The author whose PR loses closes it and
says why. Maintainers review the earlier valid claim first and only decide
when the authors can't.

## Motivation

Even with claims (RFC-0003), overlaps still happen: someone was already
working before the issue was filed, a PR fixes a bug without naming the issue,
or two people claim within minutes. Today the reviewer finds out, compares
both PRs, and closes one. That is review time spent on work that is thrown
away, at a time when review is the bottleneck (see RFC-0008).

Our experience from several hundred PRs across the craft apps: when we
compared honestly, the other PR was sometimes better, and almost always for
one of a few reasons. Writing these down makes everyone's first PR better.

## Proposal

1. **Look before opening.** Right before opening a PR, search open PRs for the
   issue number and for the function or file you changed.
2. **Compare, then decide yourself.** If another PR fixes the same thing,
   compare the two with these questions:
   - Does it fix only where the bug is? A fix limited to the one platform or
     code path that is broken beats one that changes every path.
   - Does it match how neighbouring code already handles the same case
     (same error, same empty result, same helper)?
   - Does it cover every point of the issue and every way a user reaches the
     bug (each shortcut, menu and input path), with a test for each?
   - Does it update the docs, help text or shortcut tables that describe it?
3. **The better fix stays.** If theirs is better or equal, close yours with a
   short comment that says why, and link it. If yours is clearly better (for
   example theirs fixes half the issue), say what the difference is in one
   factual sentence on both PRs. Don't argue about who was first.
4. **Combine when it makes sense.** If each PR has something the other lacks,
   one author adds the missing part to the other PR, with credit
   (`Co-authored-by`), and the other PR closes.
5. **Shared mechanisms first.** If one PR adds a general mechanism that the
   other could use (for example a common helper), the specific PR waits for
   the general one and is rebuilt on top of it.
6. **Maintainers decide ties.** When the authors don't agree, a maintainer
   picks one and says why on both PRs. By default the earlier valid claim
   (RFC-0003) is reviewed first.

## Alternatives

- **First PR wins.** Simple, but it rewards speed over quality and
  encourages opening unfinished PRs to hold a place.
- **Maintainers always decide.** This is what happens now, and it puts all the
  comparison work on the people we have the fewest of.

## Open questions

- Should a closed duplicate that had a useful idea be listed in the merged PR
  (for example "includes the approach from #X")?
