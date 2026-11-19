# RFC-0048: One shared AGENTS.md

Obsoletes: [RFC-0006](0006-coding-agents.md), [RFC-0040](0040-shared-library-first-guidance.md)

Updates: [RFC-0003](0003-claiming-issues.md), [RFC-0014](0014-accessibility.md)

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

Much of the standards text comes from earlier proposals. `standards/claims.md`
carries RFC-0003 and RFC-0004 ([craft-contrib#5](https://github.com/storytold/craft-contrib/pull/5)),
and `standards/pull-requests.md` carries RFC-0005
([craft-contrib#6](https://github.com/storytold/craft-contrib/pull/6)), all by
MarekSpodBiedry; their pull requests hold the discussion behind each rule.
`standards/accessibility.md` carries RFC-0014, by Futyn-Maker.

## Proposal

1. **The shared file.** `AGENTS.md` at the root of this repository holds the
   shared rules. It is the living document for this RFC (RFC-0013 point 5):
   wording fixes are ordinary pull requests; adding, removing or changing a
   rule needs an RFC that updates this one and changes the file in the same
   pull request.
2. **Keywords.** The file uses the key words of RFC 2119 and RFC 8174. MUST and
   MUST NOT are for legal and safety limits (clean room, licences, secrets,
   irreversible steps, reporting checks accurately). Other rules are SHOULD or
   SHOULD NOT.
   An agent that departs from a SHOULD says so and why in the PR. An operator
   may set aside a SHOULD, stated in the PR, but never a MUST.
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
   modules where `unsafe` and a one-lint `#[allow]` are allowed. It may add
   rules stricter than the shared ones, never weaker. Text there works for
   every reader: no notes about one machine, no links readers cannot open.
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
10. **Exceptions.** These differ from a shared rule in one repository, and
    that repository's `## This repository` states them:
    - craft-launcher uses `jiff` instead of `chrono` (rule 12) until
      [storytold/craft-launcher#1526](https://github.com/storytold/craft-launcher/issues/1526)
      moves it; `standards/time.md` records it.
    - craft-launcher's tests require a row in every one of its own catalogs
      for new user-facing text, where rule 16 asks for English only.

## Changes to accepted RFCs

RFC-0013 points 2 and 3: this RFC updates two accepted RFCs, and its pull
request adds the back-links.

- **RFC-0003 (claims).** An open pull request that says `Part of #N`, not
  only a draft that says `Fixes #N`, counts as a claim (`standards/claims.md`).
- **RFC-0014 (accessibility), point 4.** A kittest test is required where
  warranted rather than for every changed control: a new or changed keyboard
  path, focus move or announcement warrants one, and a control's role or name
  alone does not. The built-in name check stays on, and tests do not turn it
  off.
- **RFC-0014, point 10.** The generated `house-rules-accessibility.md` block is
  retired; rule 19 of the shared file is the short form, and the full rules stay
  in `standards/accessibility.md`.

Rule 16 also replaces the "a line in every catalog" of RFC-0005 point 5
([craft-contrib#6](https://github.com/storytold/craft-contrib/pull/6), closed
without being accepted) with English only. `standards/naming.md` keeps
RFC-0029's meaning: its edits are wording only.

## Order with RFC-0047

This pull request merges only after RFC-0047
([craft-contrib#71](https://github.com/storytold/craft-contrib/pull/71)) is
accepted: rules 25 to 29, `pull-requests.md`'s C1 exemption and
`accessibility.md` link `standards/testing.md`, which RFC-0047 adds. Rules 25
to 29 follow RFC-0047, and a change to them follows RFC-0047's process. If
RFC-0047 is rejected, rules 25 to 29 are replaced by house rule core 4's "add a
test that fails without the fix", and the links to `standards/testing.md` are
removed, before this merges.

## Rollout

1. Merge the shared file and the standards; the generator and the check follow
   in craft-common-actions.
2. Pilot in craft-libs and two apps.
3. Move each remaining repository in its own pull request. Each lists every
   rule it removes from the repository's file and where that rule now lives,
   or why it was dropped, so no guard disappears unseen.
4. Delete the house-rules files.

Every public storytold repository is in scope. Those with no `AGENTS.md` get
one: craft-common-actions, craft-fonts, craft-extensions, craft-skills,
craft-wiki, craft-launcher-assets and crafthub. craft-pr-review-tool and
craft-repo-status-app, which have only `CLAUDE.md`, get an `AGENTS.md`, and
their `CLAUDE.md` becomes `@AGENTS.md` (proposal point 7).

What each rollout pull request moves into `## This repository`:

- **Stricter local rules stay**, as proposal point 4 allows: materialcraft
  never names its reference product, also in code and commits; the corpus
  repositories ban force-pushing, because apps pin their commits; filmcraft's
  `#[allow(unsafe_code)]` on its FFI modules, and the one
  `#[allow(clippy::expect_used)]` with a reason that five apps allow, are named
  as rule 13's modules.
- **App facts that depend on paths.** An app with an agent entry point
  (`<app>-cli mcp`) states that it MUST refuse to start without
  `--automation-read-root` and `--automation-write-root` and refuse file
  access outside them (RFC-0015). An app with an append-only progress log
  names the path for a new fragment file per change (RFC-0017).
- **Exceptions** from proposal point 10.

What rollout drops or narrows:

- craft-anything's "rebase on `main` before asking for review" conflicts with
  rule 34 and is dropped.
- `deny.toml` lists match `standards/licences.md`: craft-ai-integrations'
  `MIT-0` and `bzip2-1.0.6` and pdfcraft's `MIT-0` are removed, or an RFC adds
  them first.
- `LicenseRef-Storyteller-Trademark` in attribution files becomes
  `LicenseRef-ArtCraft-Trademark` for the same marks.

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
