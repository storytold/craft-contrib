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
17, PhotoCraft 14, and FigCraft 2. PhotoCraft's generated
`docs/scorecard.md` conflicts the same way and is also out of scope. The
other parity docs have the same shape: a status line, a table, prose
around it, and a revision history. EffectCraft's `docs/gaps.md`
and `docs/ui-parity.md` are the example. The status line is a
"previously …" chain, the same failure [RFC-0017](0017-progress-doc-fragments.md)
removed from `ROADMAP.md`.

These are not a log. A log only grows. A gap or a parity row has a state
that changes, so the fragment is the item, not the change.

## Decision

- **One file per item.** `docs/gaps.d/<number>-<slug>.md`, and the same
  beside each parity doc an app keeps (`docs/ui-parity.d/`,
  `docs/file-format-parity.d/`, and the rest). The number is stable and is
  not a pull-request number; the slug is lower case, digits and `-`.
  Existing numbered gaps keep their number and references, for example
  `docs/gaps.d/18-relative-paths.md`. New items use the next free number;
  CI rejects a pull request that adds a duplicate number, even when the
  slugs differ. Reordering never changes a number. Items are never deleted.
  Two pull requests that change different items do not touch the same
  file. Two that change the same item still conflict.
- **One named field per configured column.** An item has the
  front-matter fields its table configuration names, and each is required:
  for example `read:`, `write:` and `fidelity:` for a format, or separate
  state and percent fields for a UI row. `status:` is reserved for
  open/closed (below), so a free-text state column takes another field
  name. Preserve each existing column's vocabulary,
  including values such as `partial` and `beyond`; do not force them to
  numbers. The body holds the row's evidence.
- **An item's folder is its table.** Items are found by folder, one per
  table; there is no `table:` key. A doc with one table keeps its items in
  `docs/<doc>.d/`. A doc with several tables has one subfolder per table,
  `docs/<doc>.d/<table>/` (for example `docs/file-format-parity.d/stills/`).
  The target configuration maps each folder to the heading the table sits
  under, and to which table under that heading when there are several.
- **Order is a field, not a renumber.** An item may set `rank:`, a whole
  number. The joiner sorts ranked items by rank, then the unranked items
  after them, by number. There is no `order:` key.
- **Closing an item keeps it.** A closed item sets `status: closed` (an
  open item has `status: open` or no `status:`). Each table's configuration
  says whether closed rows are struck through, every cell but the number,
  or left out. Closing does not rewrite the other files to close the hole
  in the numbering.
- **The same joiner as [RFC-0017](0017-progress-doc-fragments.md).** The
  craft-libs crate gains a table target: item folder, target file, the
  heading the table sits under (and which table under it), and the named
  column fields. It replaces that table and leaves the prose around it
  alone. CI joins nightly, so
  `main` is at most a day behind. `cargo xtask progress` prints the doc
  with the tables joined, as it does for the log.
- **An item pull request leaves the shared lines alone.** It does not
  edit the status line, the overall percent, the dimension summary, or
  the revision history. The joiner writes the status line from the
  newest item change: the commit date of the latest commit touching the
  item files that changed the table, not the item's creation date or
  filesystem mtime, and never after today. The line contains date and kind
  of change only. An item change is always `minor`: when a doc's items
  change, the joiner writes that doc's `Last updated` and `Change: minor`.
  A change to the prose assessment, or to a number that is a judgement
  rather than a cell, is its own pull request.
- **No revision-history row for an item-only change.** The item file
  already records the state. Other changes to the doc still add a row.
- **CI catches hand edits to joined tables.** `cargo xtask progress --check`
  joins the configured tables and checks the pull-request diff against their
  source items. A hand-edited joined table must have matching item changes
  and agree with the rendered result; otherwise CI fails before nightly
  joining can overwrite it. The check stops accidental and agent hand-edits,
  which are what cause the conflicts. It does not stop deliberate tampering,
  such as a pull request that disguises an edit with surrounding Markdown;
  review covers that, as it does for code. Compare protected table regions
  from the PR base to its head, mapping each changed row/cell to the
  matching source-item change and rendered value. An unrelated source
  change does not authorize a manual table edit. Source-item-only PRs
  remain valid; do not unconditionally demand that every freshly rendered
  table be committed. Report the table, row and source item needed for
  repair.
- **CI validates every item**: its file name, every configured field,
  `rank:` and `status:` values, and that no pull request adds a duplicate
  item number.
- **A generated checklist stays generated.** Where an app already builds
  `docs/parity-checklist.md` from a catalogue, pull requests change the
  catalogue. They do not hand-write fragments of the output.

## What this asks of each app

- Move the existing rows into item files once. Preserve every state cell,
  evidence, table assignment, existing row order, numeric gap identifier
  and incoming gap reference. The prose above and below the table stays
  in the doc. Compare the joined result with the original before rollout.
  Where a status line carries a running list of changes (PhotoCraft's
  `**Change:** minor (…)` with issue numbers), that list goes away when the
  joiner takes over the line.
- Inventory open PRs that edit those tables or the RFC-0017 progress log.
  Provide a scripted conversion of each row/log edit to its matching item
  or fragment and verify the joined result preserves that PR's intended
  change. Ambiguous edits require author/maintainer review; never drop
  them. This migration applies to RFC-0017 log edits as well as item rows.
  Do not enable nightly overwrites until old-format PRs are converted or
  fail safely with a specific migration diagnostic.
- Point xtask at the new table targets, and add the check to CI.
- Tell agents, in `AGENTS.md`, that an item change is a change to that
  item's file, and remove any instruction to edit a doc's status line or
  add a revision-history row for an item change ([RFC-0006](0006-coding-agents.md)).

## Rollout acceptance tests

Before any app rollout, the shared joiner and app checks demonstrate:

- A hand edit to committed joined output fails with no source change;
  matching item changes are accepted and unrelated prose is preserved.
  A partial/manual row edit still fails when a different source item also
  changes. A source-item-only PR passes without committing joined output.
- A multi-state row round-trips all configured cells without coercing
  textual values, and missing required fields fail.
- Two tables in one document receive only their own items, each from its
  own folder.
- Gap 18 remains gap 18 after migration, closure and reordering; existing
  cross-references still resolve and duplicate numbers fail.
- Ranked items sort by rank and unranked items after them by number; a
  closed item is struck through or left out as its table is configured;
  an item change writes `Change: minor`.
- Open-PR table and progress-log edits survive conversion; ambiguity fails
  visibly rather than being overwritten on the next nightly join.

The RFC-0017 forward reference in this PR lands only together with an
accepted RFC-0018, and must follow the final accepted scope. The comments
retaining these rollout requirements are linked below; approval of the
approach is not evidence that the tests or migration have already shipped.

## Settled in review

The two open questions were settled as built in
[storytold/craft-libs@997c89a](https://github.com/storytold/craft-libs/commit/997c89a4204e4dce56c3a6c972adee1995027f1c),
the head of [storytold/craft-libs#629](https://github.com/storytold/craft-libs/pull/629)
on 2026-11-06. A later change to that pull request is a difference from
this RFC, not a change to it:

- The change kind of an item change is always `minor`.
- An item without `rank:` sorts after the ranked items, by number.
  Migration must still preserve current order. Existing gap numbers are not
  renumbered.

## Not in this RFC

Progress-log storage and joining semantics remain in
[RFC-0017](0017-progress-doc-fragments.md); the migration safety requirements
above apply to both progress logs and item tables.
What the assessments say, and rewriting their hours or percentages.
Stopping the commit of a generated checklist or scorecard.

## References

- [RFC-0017](0017-progress-doc-fragments.md), and the count in its text
- EffectCraft `docs/gaps.md`, `docs/ui-parity.md`, `docs/file-format-parity.md`
- [storytold/craft-libs@997c89a](https://github.com/storytold/craft-libs/commit/997c89a4204e4dce56c3a6c972adee1995027f1c), the joiner as built
  ([storytold/craft-libs#629](https://github.com/storytold/craft-libs/pull/629))

## Review requirements carried forward

- [Benandez's four blocking requirements and migration questions](https://github.com/storytold/craft-contrib/pull/30#pullrequestreview-5798420051).
- [Skylar's approval retains all four as pre-rollout requirements](https://github.com/storytold/craft-contrib/pull/30#pullrequestreview-5798445920).
- [Benandez's four details as built in craft-libs#629, and the threat model](https://github.com/storytold/craft-contrib/pull/30#issuecomment-6689561084).
