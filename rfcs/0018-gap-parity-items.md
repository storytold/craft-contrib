# RFC-0018: Gap and parity items are files

[RFC-0017](0017-progress-doc-fragments.md) stops pull requests editing the
progress log and its status line. The larger clash is the docs that keep
one row per item in a shared table. Closing one gap rewrites that row,
the status line above it, and a revision-history row. The next pull
request does the same to a different item, and the two conflict.

On 2026-11-06, of the pull requests that conflicted with their base,
`docs/gaps.md` was one of the files in 911, `docs/ui-parity.md` in 551,
`docs/parity-checklist.md` in 434 and `docs/file-format-parity.md` in
362. The other parity docs have the same shape: a status line, a table,
prose around it, and a revision history. EffectCraft's `docs/gaps.md`
and `docs/ui-parity.md` are the example. The status line is a
"previously …" chain, the same failure [RFC-0017](0017-progress-doc-fragments.md)
removed from `ROADMAP.md`.

These are not a log. A log only grows. A gap or a parity row has a state
that changes, so the fragment is the item, not the change.

## Decision

- **One file per item.** `docs/gaps.d/<id>.md`, and the same beside each
  parity doc an app keeps (`docs/ui-parity.d/`, `docs/file-format-parity.d/`,
  and the rest). `<id>` is a stable slug for that item. It is not a rank
  and not a pull-request number. Two pull requests that change different
  items do not touch the same file. Two that change the same item still
  conflict.
- **`status:` is required.** The values are the ones that table already
  uses: `open` or `closed` for a gap, the fidelity word for a format, a
  percent where the column is a percent. The rest of the file is the
  evidence that cell holds today.
- **Order is a field, not a renumber.** A gap file may set `order:`. The
  joiner sorts by it. Closing a gap sets `status: closed` and does not
  rewrite the other files to close the hole in the numbering.
- **The same joiner as [RFC-0017](0017-progress-doc-fragments.md).** The
  craft-libs crate gains a table target: fragment folder, target file,
  the heading the table sits under, and the columns. It replaces that
  table and leaves the prose around it alone. CI joins nightly, so
  `main` is at most a day behind. `cargo xtask progress` prints the doc
  with the tables joined, as it does for the log.
- **An item pull request leaves the shared lines alone.** It does not
  edit the status line, the overall percent, the dimension summary, or
  the revision history. The joiner writes the status line from the
  newest item: date and kind of change only. A change to the prose
  assessment, or to a number that is a judgement rather than a cell, is
  its own pull request.
- **No revision-history row for an item-only change.** The item file
  already records the state. Other changes to the doc still add a row.
- **CI** fails a file with no `status:`, an empty body, or a duplicate id.
- **A generated checklist stays generated.** Where an app already builds
  `docs/parity-checklist.md` from a catalogue, pull requests change the
  catalogue. They do not hand-write fragments of the output.

## What this asks of each app

- Move the existing rows into item files once. The prose above and below
  the table stays in the doc.
- Point xtask at the new table targets, and add the check to CI.
- Tell agents, in `AGENTS.md`, that an item change is a change to that
  item's file (RFC-0006).

## Not in this RFC

The progress log, which [RFC-0017](0017-progress-doc-fragments.md) covers.
What the assessments say, and rewriting their hours or percentages.
Stopping the commit of a generated checklist.

## References

- [RFC-0017](0017-progress-doc-fragments.md), and the count in its text
- EffectCraft `docs/gaps.md`, `docs/ui-parity.md`, `docs/file-format-parity.md`
