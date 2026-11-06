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

On 2026-11-06, 5,385 of 9,915 open pull requests across 38 repositories
conflicted with their base. 1,373 of those touch `ROADMAP.md`. 109
conflict only there: that is what this RFC fully unblocks. In the other
1,264 the `ROADMAP.md` conflict goes away and the rest remain. Most
docs-only conflicts are `docs/gaps.md` and the parity docs, which are a
follow-up RFC, not this one. `CHANGELOG` did not show up as a conflict.

## Decision

- **Status line.** The joiner writes it, from the newest fragment: date
  and kind of change only, for example `**Last updated:** 2026-11-03 ·
  **Change:** minor`. No description and no "previously" chain. A pull
  request does not edit the status line. Two open pull requests would
  otherwise write different dates on the same line and still conflict.
- **Progress-log entries are fragments.** A pull request adds one file,
  `docs/progress.d/<issue>-<branch-slug>.md`, holding its entry exactly
  as it reads in the log. An app keeps its own entry format: a bullet, a
  table row. The issue number alone is not an id: two pull requests for
  one issue would both add `<issue>.md` and conflict. The pull request
  number is not known when the branch is written. CI still rejects a
  duplicate id. Pull requests no longer edit `## Progress log` directly.
- **No revision-history row for a log-only change.** The log already
  records it. Other changes to a doc still add a row.
- **One joiner, in craft-libs.** A small crate, maintained by the
  craft-libs maintainers, built from MailCraft's
  `xtask/src/release_notes.rs` (about 150 lines). Each app's xtask
  depends on it. Copying it into every app would drift. It is configured
  per target as (fragment folder, target file, anchor heading, order)
  and serves release notes and the progress log. Revision rows can use
  it later if conflicts on them remain.
- **Reading and joining.** The committed log and the unjoined fragments
  together are the doc. `cargo xtask progress` prints them joined. CI
  joins nightly, so `main` is at most a day behind. A maintainer's
  `--write` moves the fragments under their heading, newest first by
  commit date, and deletes them. Only the joiner touches the log heading
  and the status line, so the join commit does not conflict with pull
  requests. Anything that reads `ROADMAP.md` raw, including the site
  generator, calls the joiner. That change lands before rollout.
- **CI** (`cargo xtask ci`) fails a pull request whose fragment does not
  join: bad format, an empty file, a duplicate id.

## What this asks of each app

- Clean up the status line once, to the new form (WordCraft: rewrite line
  7 and delete the duplicate line 8). After that, pull requests leave it
  alone.
- Depend on the craft-libs joiner from xtask, and add the check to CI.
- Write new log entries as fragments. Existing entries stay where they are.
- Tell agents about the change in `AGENTS.md` (RFC-0006).

## Not in this RFC

What the progress docs say. Release-note fragments, which MailCraft
already has. `docs/gaps.md` and the parity docs: they track the state of
each item rather than appending entries. [RFC-0018](0018-gap-parity-items.md)
gives them one file per item, with a `status:` field.

## References

- MailCraft#464 (release-note fragments), MailCraft#495 (status lines),
  MailCraft#759, MailCraft#776
- MailCraft `xtask/src/release_notes.rs`,
  `packaging/release-notes.d/README.md`, `docs/releasing.md`
- RFC-0006 (`AGENTS.md`)
