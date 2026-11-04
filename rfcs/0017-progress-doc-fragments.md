# RFC-0017: Progress-doc entries are fragments

Every pull request edits the same few lines of `ROADMAP.md` and the other
progress docs, so any two open PRs conflict there. In WordCraft the
status line holds a "previously …" chain of 100+ entries, and the line
is duplicated, likely after a bad merge. Its revision history is out of
date order. Since 2026-10-04 WordCraft has 75 `Merge main` commits and
MailCraft 146. Not all of those are docs-only, but MailCraft#759 and
MailCraft#776 were, on 2026-11-03.

MailCraft fixed the same problem for release notes (MailCraft#464) and
status lines (MailCraft#495). This RFC applies the same fix to the
progress docs in every app.

## Decision

- **Status line:** date and kind of change only, e.g.
  `**Last updated:** 2026-11-03 · **Change:** minor`. No description, no
  "previously" chain. Two PRs on the same day write the same line, so git
  merges it. What changed goes in the doc's log or revision history.
- **Progress-log entries are fragments.** A PR adds one file,
  `docs/progress.d/<id>.md`, holding its entry exactly as it reads in the
  log (an app keeps its own entry format: a bullet, a table row). `<id>` is
  the issue number, else the PR number, else a short name. PRs no longer
  edit `## Progress log` directly.
- **No revision-history row for a log-only change.** The log already
  records it. Other changes to a doc still add a row.
- **One joiner.** MailCraft's `xtask/src/release_notes.rs` (about 150
  lines) becomes a generic fragment joiner, configured per target as
  (fragment folder, target file, anchor heading, order). It serves release
  notes and the progress log, and revision rows later if needed.
- **Reading and joining.** `cargo xtask progress` prints the doc with the
  fragments joined. A maintainer's `--write` moves the fragments under
  their heading, newest first by commit date, and deletes them. It runs at
  least before every release, and whenever a maintainer wants `main` tidy.
  Only the joiner touches the log heading, so the join commit doesn't
  conflict with PRs.
- **CI** (`cargo xtask ci`) fails a PR whose fragment doesn't join: bad
  format, an empty file, a duplicate id.

## What this asks of each app

- Clean up the status line once, to the new form (WordCraft: rewrite line
  7 and delete the duplicate line 8).
- Add the joiner to its xtask and the check to its CI.
- Write new log entries as fragments. Existing entries stay where they are.
- Tell agents about the change in `AGENTS.md` (RFC-0006).

## Not in this RFC

What the progress docs say; release-note fragments, which MailCraft
already has; `docs/gaps.md` and the parity docs, which may need the same
treatment later.

## Open

- Where the joiner lives: craftrules, a crate in craft-libs, or copied into
  each app's xtask. Copied is simplest; shared avoids drift.
- Can `ROADMAP.md` on `main` lag until the next join? People, agents and
  any site or readiness scripts that read it would see only joined entries
  unless they run `cargo xtask progress`. Something that reads it raw
  (craftrules `scripts/*`, the site generator) must be checked first.
- Should the join be scheduled (nightly, by CI) instead of by hand?
- Should revision-history rows become fragments too
  (`docs/revisions.d/<doc>/<id>.md`), if conflicts on them remain?
- craftrules `standards/progress-docs.md` must change to match. It
  couldn't be read while drafting this; its owner should confirm the two
  agree.
- Measure how many of the `Merge main` commits were for progress docs
  alone.

## References

- MailCraft#464 (release-note fragments), MailCraft#495 (status lines),
  MailCraft#759, MailCraft#776
- MailCraft `xtask/src/release_notes.rs`,
  `packaging/release-notes.d/README.md`, `docs/releasing.md`
- craftrules `standards/progress-docs.md`
- RFC-0006 (`AGENTS.md`)
