# RFC-0048: One shared AGENTS.md

Obsoletes: [RFC-0006](0006-coding-agents.md), [RFC-0040](0040-shared-library-first-guidance.md)

Depends on: RFC-0047 (`standards/testing.md`), [craft-contrib#71](https://github.com/storytold/craft-contrib/pull/71)

## Summary

Every Craft repository's `AGENTS.md` starts with one shared text, kept in
`AGENTS.md` at the root of this repository. Each repository adds a
`## This repository` section that describes the project: layout, commands,
paths and conventions. Long topics live in linked standards. The generated
house-rules blocks and their source files are retired.

## Motivation

RFC-0006 put a 3 KB generated block into each repository and left the rest to
local text. The local text grew: across 28 public repositories, the same rule
appears in up to 20 wordings, some repositories relax rules others enforce, and
agent rules sit in open RFC pull requests that never merged. An agent that
moves between repositories meets different rules for the same situation, and a
weak model follows whichever sentence it read last.

One text, the same in every repository, gives every agent the same rules, lets
one review cover them, and makes a rule change one pull request.

## Proposal

1. **The shared file.** `AGENTS.md` at the root of this repository holds the
   shared rules. It is the living document for this RFC (RFC-0013 point 5):
   wording fixes are ordinary pull requests; adding, removing or changing a
   rule needs an RFC that updates this one and changes the file in the same
   pull request.
2. **Keywords.** The file uses the key words of RFC 2119 and RFC 8174. MUST and
   MUST NOT are for legal and safety limits (clean room, licences, secrets,
   irreversible steps, automation roots, reporting checks accurately). Other
   rules are SHOULD or SHOULD NOT. An agent that departs from a SHOULD says so
   and why in the PR.
3. **Linked standards.** `standards/claims.md`, `pull-requests.md`,
   `clean-room.md`, `licences.md`, `accessibility.md`, `naming.md`, `time.md`
   and `testing.md` hold the detail a rule needs on some tasks. `testing.md`
   comes from RFC-0047; this RFC depends on it. The shared file states every
   rule an agent needs on every task and links the standard for the rest. A
   standard adds detail and exceptions to the rule that links it, and an
   exception written in a standard applies; on any other conflict the shared
   file wins.
4. **Repository files.** Each repository's root `AGENTS.md` is the shared
   text, copied between markers, then `## This repository`, which describes
   that project: layout, commands, paths and conventions, including the
   modules where `unsafe` is allowed. Text there works for every reader: no
   notes about one machine, no links readers cannot open.
5. **Integrity check.** The copy sits between
   `<!-- craft-agents:start source=<full merged commit SHA> sha256=<digest> -->`
   and `<!-- craft-agents:end -->`. A check in `craft-common-actions` compares
   the copy with the shared file at the recorded revision and checks that the
   revision is in this repository's merged history. An edited copy or an
   invalid revision fails; an intact older copy warns that it is stale. Text
   and newline normalization are the same in the generator and the check.
6. **Updates.** A maintainer-run command, later an automated job, opens one
   update pull request per repository with the new copy, the source revision
   and a one-line summary of the rule changes. It changes nothing outside the
   markers.
7. **Claude Code.** Where `CLAUDE.md` exists, it contains `@AGENTS.md`, and
   nothing else except rules specific to that tool.
8. **Size.** The shared part stays under 8 KB, and `## This repository` under
   4 KB, measured in bytes. A rule that only some tasks need goes to a
   standard first.
9. **Retired.** `standards/house-rules-core.md`, `house-rules-rust.md` and
   `house-rules-accessibility.md` are deleted once every repository carries
   the shared copy. The agent rules in craft-contrib#5, #6, #33, #45 and #57 are
   in the shared file or the standards; those pull requests keep only what is
   not an agent rule.

## Rollout

1. Merge the shared file and the standards; the generator and the check follow
   in craft-common-actions.
2. Pilot in craft-libs and two apps.
3. Move each remaining repository in its own pull request. Each lists every
   rule it removes from the repository's file and where that rule now lives,
   or why it was dropped, so no guard disappears unseen.
4. Delete the house-rules files.

## Alternatives

- **Keep RFC-0006's core block and local rules.** That is today's state, with
  the drift described above.
- **Link the shared file instead of copying it.** Agents do not reliably open
  links, so a rule an agent needs on every task has to be in the file it
  reads.
- **A longer shared file that holds every rule.** It costs tokens on every
  task and buries the rules that matter on all of them; detail an agent needs
  only sometimes lives in the standards, each linked from its rule.
- **Per-repository exceptions approved by a maintainer.** Each exception is a
  rule that differs by repository, which is what this RFC removes. Where a
  repository needs a different fact (a command, an `unsafe` module), it states
  it in `## This repository`.

## Questions for the team

1. Clean room: confirm with whoever handles legal questions that a screen-level
   behaviour specification, written by an agent that only operates the app, is
   outside the "monitoring inputs and outputs" wording of the Adobe General
   Terms §17(B).
2. Fixtures: third-party published files are committed only under a licence on
   the allowlist, otherwise fetched from a manifest. Is that the right reading?
3. Corpus threshold: `testing.md` sends binary fixtures over 1 MB to a corpus
   repository. Is 1 MB the right number?
4. craft-launcher keeps `jiff` until it moves to `chrono`; the migration issue
   will be linked from `time.md`. Do you accept that?
5. Review response time: 3 days for actionable requests, with a reminder at 7
   and eligibility for closure at 14 (RFC-0006 had 7, 14 and 30).
6. Size limit: `standards/pull-requests.md` splits a PR that passes 400 changed
   lines outside generated files, fixtures and pure moves. Is 400 the right
   number?
7. Merge craft-contrib#53?
8. Decide patent-pool encoders (H.264, HEVC, AAC) with RFC-0031?
