# RFC-0019: RFC scope and affected-app review invitations

Updates: [RFC-0000](0000-decision-process.md)

## Summary

Use RFCs for cross-app architecture, policy and governance decisions, not
ordinary implementation PRs or minor changes that leave those decisions
unchanged. When starting a comment period, explicitly invite maintainers of
affected apps to weigh in.

## Motivation

[Skylar requested both clarifications on RFC-0000](https://github.com/storytold/craft-contrib/pull/1#issuecomment-6206151051),
and [echelon endorsed them](https://github.com/storytold/craft-contrib/pull/1#issuecomment-6235760452).
The accepted text did not incorporate them. This proposal puts the missing
clarifications through the decision process rather than treating discussion
comments as adopted policy.

## Proposal

1. **RFC scope.** RFCs are for cross-app architecture, policy and governance
   decisions, including changes to this decision process. Ordinary
   implementation PRs and minor changes that do not change those decisions
   use the ordinary PR process. A small diff that changes such a decision
   still needs an RFC.

2. **Affected-app invitations.** The maintainer who starts an RFC's comment
   period explicitly invites maintainers of the affected apps to weigh in on
   the PR when announcing the period's end date. This does not require
   unanimous approval or an affirmative response from every invitee.

All other RFC-0000 rules remain unchanged. Anyone can open an RFC or comment.
After the comment period, approval must come from a maintainer other than the
author, and no maintainer's objection may remain unresolved. An owner can
decide an objection, or merge or close an RFC early, and says why on the PR.

## Adoption

This is a proposed process change, not an effective policy clarification.
Under RFC-0000 it needs a seven-day comment period started by a maintainer
announcing its end date, followed by the required approval and objection
resolution, unless an owner uses RFC-0000's explained early-decision path.
Opening this draft and time spent discussing RFC-0000 do not start or satisfy
that period.

[RFC-0013](https://github.com/storytold/craft-contrib/pull/17) separately proposes
how accepted RFCs are amended. It remains a proposal; this RFC does not depend
on treating it as adopted. Before accepting this RFC, reconcile its amendment
format with any process changes accepted in the meantime.

This proposal does not change reviewer eligibility, require a non-committer
reviewer, or change branch protection.

