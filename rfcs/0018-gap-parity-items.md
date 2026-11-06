# RFC-0018: Gap and parity items are files

Updates: [RFC-0017](0017-progress-doc-fragments.md) (open-PR migration).

[RFC-0017](0017-progress-doc-fragments.md) stops pull requests editing the
progress log and its status line. The larger clash is the docs that keep
one row per item in a shared table. Closing one gap rewrites that row,
the status line above it, and a revision-history row. The next pull
request does the same to a different item, and the two conflict.

On 2026-11-06, of the pull requests that conflicted with their base,
`docs/gaps.md` was one of the files in 911, `docs/ui-parity.md` in 551,
`docs/parity-checklist.md` in 434 and `docs/file-format-parity.md` in
362. Of the 434 checklist conflicts, 343 were in LightCraft's handwritten
checklist and are covered here. The other 91 were in generated checklists
and remain out of scope: CADCraft 19, DeckCraft 24, GridCraft 15, WordCraft
17, PhotoCraft 14, and FigCraft 2. The other parity docs have the same
shape: a status line, a table,
prose around it, and a revision history. EffectCraft's `docs/gaps.md`
and `docs/ui-parity.md` are the example. The status line is a
"previously …" chain, the same failure [RFC-0017](0017-progress-doc-fragments.md)
removed from `ROADMAP.md`.

These are not a log. A log only grows. A gap or a parity row has a state
that changes, so the fragment is the item, not the change.

## Decision

- **One file per item.** `docs/gaps.d/<id>.md`, and the same beside each
  parity doc an app keeps (`docs/ui-parity.d/`, `docs/file-format-parity.d/`,
  and the rest). `<id>` is stable and is not a pull-request number.
  Existing numbered gaps keep their number and references, for example
  `docs/gaps.d/18-relative-paths.md`. New gaps use the next free number;
  CI rejects duplicate numeric gap identifiers even when their slugs differ.
  Other items keep a stable identifier. Reordering never changes an id.
  Two pull requests that change different items do not touch the same file. Two that change the same item still
  conflict.
- **One named field per configured state column.** An item has the
  front-matter fields its table configuration names: for example `read:`,
  `write:` and `fidelity:` for a format, or separate `status:` and `percent:`
  fields for a UI row. `status:` alone is the gap-table case, not a schema
  imposed on every parity row. Preserve each existing column's vocabulary,
  including values such as `partial` and `beyond`; do not force them to
  numbers. The body holds the row's evidence.
- **Each item identifies its table.** A required `table:` field selects a
  stable table identifier in the target configuration. That identifier maps
  to one anchor heading in the document. A document can configure several
  tables in the same fragment folder; CI rejects missing, unknown or
  ambiguous table targets rather than putting a row in the wrong table.
- **Order is a field, not a renumber.** A gap file may set `order:`. The
  joiner sorts by it. Closing a gap sets `status: closed` and does not
  rewrite the other files to close the hole in the numbering.
- **The same joiner as [RFC-0017](0017-progress-doc-fragments.md).** The
  craft-libs crate gains a table target: fragment folder, target file,
  stable table identifier, the heading the table sits under, and the named
  column fields. It replaces that table and leaves the prose around it
  alone. CI joins nightly, so
  `main` is at most a day behind. `cargo xtask progress` prints the doc
  with the tables joined, as it does for the log.
- **An item pull request leaves the shared lines alone.** It does not
  edit the status line, the overall percent, the dimension summary, or
  the revision history. The joiner writes the status line from the
  newest item change: the commit date of the latest commit touching that
  target's item files, not the item's creation date or filesystem mtime.
  The line contains date and kind of change only; the change-kind source
  must be settled in the open questions below before rollout. A change to
  the prose assessment, or to a number that is a judgement rather than a cell, is
  its own pull request.
- **No revision-history row for an item-only change.** The item file
  already records the state. Other changes to the doc still add a row.
- **CI protects the source of truth.** `cargo xtask progress --check` joins
  the configured tables and checks the pull-request diff against their
  source items. A hand-edited joined table must have matching item changes
  and agree with the rendered result; otherwise CI fails before nightly
  joining can overwrite it. Compare protected table regions from the PR
  base to its head, mapping each changed row/cell to the matching source-item
  change and rendered value. An unrelated source change does not authorize
  a manual table edit. Source-item-only PRs remain valid; do not
  unconditionally demand that every freshly rendered table be committed. Report the table, row and source
  item needed for repair.
- **CI validates every configured field**, the table identifier, a
  nonempty evidence body and unique item identifiers. For numbered gaps,
  uniqueness is checked by the preserved number as well as the full id.
- **A generated checklist stays generated.** Where an app already builds
  `docs/parity-checklist.md` from a catalogue, pull requests change the
  catalogue. They do not hand-write fragments of the output.

## What this asks of each app

- Move the existing rows into item files once. Preserve every state cell,
  evidence, table assignment, existing row order, numeric gap identifier
  and incoming gap reference. The prose above and below the table stays
  in the doc. Compare the joined result with the original before rollout.
- Inventory open PRs that edit those tables or the RFC-0017 progress log.
  Provide a scripted conversion of each row/log edit to its matching item
  or fragment and verify the joined result preserves that PR's intended
  change. Ambiguous edits require author/maintainer review; never drop
  them. This migration applies to RFC-0017 log edits as well as item rows.
  Do not enable nightly overwrites until old-format PRs are converted or
  fail safely with a specific migration diagnostic.
- Point xtask at the new table targets, and add the check to CI.
- Tell agents, in `AGENTS.md`, that an item change is a change to that
  item's file (RFC-0006).

## Rollout acceptance tests

Before any app rollout, the shared joiner and app checks demonstrate:

- A hand edit to committed joined output fails with no source change;
  matching item changes are accepted and unrelated prose is preserved.
  A partial/manual row edit still fails when a different source item also
  changes. A source-item-only PR passes without committing joined output.
- A multi-state row round-trips all configured cells without coercing
  textual values, and missing required fields fail.
- Two tables in one document receive only their own items; unknown or
  ambiguous table identifiers fail.
- Gap 18 remains gap 18 after migration, closure and reordering; existing
  cross-references still resolve and duplicate numeric ids fail.
- Open-PR table and progress-log edits survive conversion; ambiguity fails
  visibly rather than being overwritten on the next nightly join.

The RFC-0017 forward reference in this PR lands only together with an
accepted RFC-0018, and must follow the final accepted scope. The comments
retaining these rollout requirements are linked below; approval of the
approach is not evidence that the tests or migration have already shipped.

## Open questions that block rollout

- Where does the status line's `minor`/`major` change kind come from for
  persistent item edits: explicit item/change metadata, or a documented
  fixed classification? The existing text did not specify this. Do not
  infer it from an item filename or silently default it in the joiner.
- What is the deterministic default position of an item without `order:`?
  Preserve current order during migration, but agree and test the default
  for new items before rollout. Do not renumber existing gap references.

## Not in this RFC

Progress-log storage and joining semantics remain in
[RFC-0017](0017-progress-doc-fragments.md); the migration safety requirements
above apply to both progress logs and item tables.
What the assessments say, and rewriting their hours or percentages.
Stopping the commit of a generated checklist.

## References

- [RFC-0017](0017-progress-doc-fragments.md), and the count in its text
- EffectCraft `docs/gaps.md`, `docs/ui-parity.md`, `docs/file-format-parity.md`

## Review requirements carried forward

- [Benandez's four blocking requirements and migration questions](https://github.com/storytold/craft-contrib/pull/30#pullrequestreview-5798420051).
- [Skylar's approval retains all four as pre-rollout requirements](https://github.com/storytold/craft-contrib/pull/30#pullrequestreview-5798445920).
