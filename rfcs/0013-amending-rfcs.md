# RFC-0013: Amending an accepted RFC

Updates: [RFC-0000](0000-decision-process.md)

## Summary

An accepted RFC is not edited. A later RFC changes it and says so under its
title, with `Updates:` or `Obsoletes:`, as [IETF RFCs](https://www.rfc-editor.org/rfc/rfc7322#section-4.1.4)
do. The only change an accepted RFC gets is a line under its title pointing to
the RFC that updates or obsoletes it, and fixes that don't change its meaning.
An RFC can name a separate document that holds the current rules, so nobody has
to read an RFC and all its amendments to find them.

## Motivation

[RFC-0000](0000-decision-process.md) says decisions are made through RFCs that
add `rfcs/NNNN-short-title.md`, but not what happens when an accepted RFC needs
to change. Editing it in place keeps the file current but loses the record: the
comment period and the approval were given to the old text, and only the git
history shows what was decided when. Rules that apps adopt one by one, such as
shared conventions, will change as they meet real code, so this will come up
soon.

## Proposal

1. **An accepted RFC's text doesn't change,** except as in points 3 and 4.
2. **A change is a new RFC** that says under its title which RFC it changes:
   - `Updates: RFC-NNNN` changes part of it; the rest still stands.
   - `Obsoletes: RFC-NNNN` replaces it entirely.

   It follows RFC-0000's process, with the comment period the changed RFC
   needed: 7 days if that one is about adding or removing an app or about this
   process, 3 days otherwise.
3. **The same PR adds a back-link** under the old RFC's title:
   `Updated by: RFC-MMMM` or `Obsoleted by: RFC-MMMM`. Someone reading the old
   RFC must be able to see that it is no longer the whole story.
4. **Fixes that don't change the meaning,** such as typos, broken links and
   formatting, go in ordinary PRs, with a maintainer's approval and no comment
   period. A fix that changes the meaning needs an `Updates:` RFC.
5. **An RFC can name a living document,** such as `docs/<topic>.md`, that holds
   the current rules it sets. Wording fixes there are ordinary PRs, as in point
   4. Adding, removing or changing a rule needs an RFC that updates the one
   naming the document, and that RFC's PR changes the document too.

This RFC updates RFC-0000, so its PR adds `Updated by: RFC-0013` to RFC-0000.

## Alternatives

- **Edit accepted RFCs in place.** The file is always current, but a reader
  can't tell which parts were reviewed and accepted, and which were changed
  later with less scrutiny, without reading the git history.
- **Version numbers inside each RFC** (v1, v2, …). The same problem: one file
  holds several decisions, and the reader has to work out which one is in
  effect.
- **Keep the back-links in an index instead of the RFC**, as the IETF does in
  its [RFC index](https://www.rfc-editor.org/rfc-index.html). That works when
  everyone reads the index first. Here people open the file from a link, so the
  pointer has to be in the file.

## Open questions

- Should obsoleted RFCs move to a separate folder? This RFC proposes not, so
  links to them keep working.
- The IETF also keeps a list of reported errors (errata). Point 4 covers errors
  that don't change the meaning, and an `Updates:` RFC covers the rest. Is a
  separate list needed?
