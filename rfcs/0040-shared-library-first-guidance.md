# RFC-0040: Shared-library-first guidance for Craft apps

Obsoleted by: [RFC-0048](0048-shared-agents-md.md)

Updates: [RFC-0006](0006-coding-agents.md)

## Summary

Add one app-scoped rule to the canonical core house rules: before implementing
reusable formats, codecs, protocols or UI, check craft-libs and relevant open
work, then reuse or extend the shared API instead of keeping another app-local
implementation. App-specific policy and model mapping remain in the app.

## Motivation

Separate guidance paragraphs in app AGENTS.md files can drift or contradict
existing rules. One canonical rule, copied by the existing generated-block
mechanism, makes the shared-library direction consistent without treating a
new instruction as evidence that any existing cutover is complete.

## Decision

Add point 16 to `standards/house-rules-core.md`. Its scope is Craft apps even
though the core block is also present in shared repositories. It preserves
behavior, hostile-input/resource limits, attribution, layering and regression
coverage. It does not authorize sibling-application crate dependencies, alter
license/source allowlists or change platform requirements.

Existing maintainer-approved exceptions remain explicit local instructions.
For example, an app-local parser selected by an accepted ADR is not silently
replaced by this guidance; changing it requires the separate decision, code
change and parity verification appropriate to that repository. A documented
exception is not a reason to silently drop behavior during a cutover.

This changes contributor guidance, not application code or dependency pins.
It does not declare any app or format migrated. The existing runtime-migration
proposal [RFC-0036](https://github.com/storytold/craft-contrib/pull/56) addresses
specific copies and their test suites; this RFC neither accepts its inventory
nor replaces its migration decisions.

## Rollout

After the canonical change is merged, regenerate each app's applicable shared
blocks from the merged source revision using the existing house-rules tool.
Do not hand-edit generated blocks or stamp an unmerged source revision.
Update existing guidance PRs rather than creating duplicate synchronization
PRs. Remove redundant app-local copies of the common paragraph, while
preserving local exceptions and app-specific restrictions.

Use the house-rules integrity check on each regenerated app file and review
its full diff for unintended changes. No runtime validation is implied by the
documentation check; code cutovers still need their own tests and reviews.

## Process

This adds a rule to the living document named by RFC-0006 and therefore follows
[RFC-0013](0013-amending-rfcs.md), including this RFC-0006 back-link. Under
RFC-0000 the normal comment period is three days; the maintainer/owner process
remains unchanged.
