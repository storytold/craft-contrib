# RFC-0080: Extensions offered on an open pull request

Updates: [RFC-0072](0072-shared-agents-md.md)

## Problem

`standards/claims.md` explains how to handle independent and duplicate pull
requests, but not a contributor offering an extension on top of someone else's
open pull request. It does not say whether the offer is a duplicate, who decides
whether to take it, or where work beyond the original issue belongs. RFC-0004's
discussion raised this case, but the rule did not make it into the shared
standard when RFC-0072 consolidated it. Helen Kwok raised the gap again after
RFC-0072 merged
([discussion](https://github.com/storytold/craft-contrib/pull/72#issuecomment-7039775151)).

## Proposal

Add this paragraph after point 2 of "When two pull requests fix the same thing"
in `standards/claims.md`:

> An extension offered on top of someone else's open pull request is not, by
> itself, a duplicate; the receiving author decides whether to take it. The
> original pull request SHOULD stay focused on the issue it fixes. Work beyond
> that issue SHOULD be a separate, focused follow-up pull request after the
> original merges. Credit the original contributor when their work is carried
> over, using `Co-authored-by` where appropriate.

This makes no change to claim priority or to the rules for competing fixes.

## Affects

All Craft repositories and contributors; `standards/claims.md` applies to
issues and pull requests across the suite.

## Open questions

None.
