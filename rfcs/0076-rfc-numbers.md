# RFC-0076: An RFC's number is its pull request's number

Updates: [RFC-0000](0000-decision-process.md)

## Summary

An RFC's number is the number of the pull request that proposes it, as in
[Rust's RFC process](https://github.com/rust-lang/rfcs). The number is known
as soon as the pull request is open, and it never changes.

## Motivation

The README tells authors to pick the next free number when they open an
RFC. Authors open RFCs at the same time, so numbers collide: two open PRs
are titled RFC-0001 and two RFC-0041, and RFC-0024 and RFC-0025 were
renumbered after a collision. Each collision costs a rename and comments on
both PRs.

## Proposal

1. **The number is the PR number.** An author opens the PR with
   `rfcs/NNNN-short-title.md`, then renames the file and sets `RFC-NNNN` in
   its heading and the PR title to the PR's number, padded to four digits.
   If the author doesn't, the maintainer who merges it does, in a last
   commit on the PR's branch.
2. **Merged RFCs keep their numbers.** Numbers are not contiguous: pull
   requests that are not RFCs, and RFCs that don't merge, use numbers too.
3. **Open RFCs move to their PR numbers,** except the four whose PR number
   a merged RFC already uses: #15, #23 and #34 keep the numbers in their
   titles (RFC-0011, RFC-0016 and RFC-0019), and #30 takes RFC-0044, since
   RFC-0018 is open PR #18's number. Each of these numbers belongs to an
   issue, so no pull request can take it. No later pull
   request can collide, because pull request numbers are already past every
   merged RFC's number.

This RFC updates RFC-0000 and changes the README's "Open an RFC" paragraph,
so its PR adds `Updated by: RFC-0076` to RFC-0000 and edits the README.

## Alternatives

- **Keep picking the next free number.** Collisions continue.
- **The maintainer gives the next free number at merge.** No collisions,
  but an open RFC has no number to refer to, and every merge needs a rename
  commit.
