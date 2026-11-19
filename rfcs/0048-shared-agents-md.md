# RFC-0048: One shared AGENTS.md

Obsoletes: [RFC-0006](0006-coding-agents.md), [RFC-0040](0040-shared-library-first-guidance.md)

Updates: [RFC-0003](0003-claiming-issues.md), [RFC-0014](0014-accessibility.md)

Depends on: RFC-0047 (`standards/testing.md`), [craft-contrib#71](https://github.com/storytold/craft-contrib/pull/71)

## Summary

Every Craft repository's `AGENTS.md` starts with one shared text, kept in
`standards/app-agents.md` in this repository. Each repository adds a
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

1. **The shared file.** `standards/app-agents.md` in this repository holds the
   shared rules. It is for the Craft app and library repositories; this
   repository's own `AGENTS.md` is a short note for work here and is not copied. It is the living document for this RFC (RFC-0013 point 5):
   wording fixes are ordinary pull requests; adding, removing or changing a
   rule needs an RFC that updates this one and changes the file in the same
   pull request. Whoever runs an agent answers for what it opens and for its
   review comments.
2. **Keywords.** The file uses the key words of RFC 2119 and RFC 8174. MUST and
   MUST NOT are for legal and safety limits (clean room, licences, secrets,
   irreversible steps, automation roots, reporting checks accurately). Other
   rules are SHOULD or SHOULD NOT. An agent that departs from a SHOULD says so
   and why in the PR.
3. **Linked standards.** `standards/claims.md`, `pull-requests.md`,
   `clean-room.md`, `licences.md`, `accessibility.md`, `naming.md`, `time.md`
   and `testing.md` hold the detail a rule needs on some tasks. `testing.md`
   comes from RFC-0047; this RFC merges only after RFC-0047 is accepted, and
   rules 27-32 follow RFC-0047's short form. If RFC-0047 is rejected, rules
   27-32 become "A bug fix SHOULD add a test that fails without it" and the
   links to `testing.md` go. The shared file states every
   rule an agent needs on every task and links the standard for the rest. A
   standard adds detail and exceptions to the rule that links it, and an
   exception written in a standard applies; on any other conflict the shared
   file wins. `time.md` records one exception: craft-launcher uses `jiff`
   until it moves to `chrono` ([craft-launcher#1526](https://github.com/storytold/craft-launcher/issues/1526)).
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
   the shared copy.
10. **Sources.** The shared file and the standards carry the agent rules of
    accepted RFCs and of four closed proposals, whose discussions explain
    them:
    - `claims.md`: RFC-0003 and RFC-0004
      ([craft-contrib#5](https://github.com/storytold/craft-contrib/pull/5)),
      by MarekSpodBiedry;
    - `pull-requests.md`: RFC-0005
      ([craft-contrib#6](https://github.com/storytold/craft-contrib/pull/6)),
      by MarekSpodBiedry;
    - rule 15:
      [craft-contrib#45](https://github.com/storytold/craft-contrib/pull/45)
      and [craft-contrib#44](https://github.com/storytold/craft-contrib/issues/44),
      by PhilC58;
    - the MPL-2.0 section of `licences.md`:
      [craft-contrib#57](https://github.com/storytold/craft-contrib/pull/57),
      by FredApps.

    [craft-contrib#33](https://github.com/storytold/craft-contrib/pull/33)
    (house-rule exceptions and contributor credits) is not carried as a
    whole: rule 20 allows contributor credits a person consented to, and
    `## This repository` holds no rules of its own.
11. **Changes to accepted RFCs.**
    - RFC-0003: any open pull request that says `Fixes #N` or `Part of #N`
      counts as a claim, and `claims.md` adds the rules for two pull requests
      that fix the same thing.
    - RFC-0014: point 4's kittest test is needed where `accessibility.md`
      says it is warranted, not for every changed control. Point 10's short
      form is retired; rule 21 of the shared file replaces it.

## Rollout

1. Merge the shared file and the standards; the generator and the check follow
   in craft-common-actions.
2. Pilot in craft-libs and two apps.
3. Move each remaining repository in its own pull request. In scope are the
   storytold repositories craft-libs, craft-launcher, craft-launcher-assets,
   craft-i18n, craft-fonts, craft-common-actions, craft-extensions,
   craft-skills, craft-wiki, crafthub, craft-ai-integrations, craft-anything,
   craft-pr-review-tool, craft-repo-status-app, indd-utils, photocraft,
   photocraft-corpus, pdfcraft, lightcraft, filmcraft, vectorcraft,
   designcraft, deckcraft, wordcraft, gridcraft, figcraft, cadcraft,
   cadcraft-corpus, soundcraft, texturecraft, materialcraft, effectcraft,
   effectcraft-corpus, anicraft and autocraft, and every new Craft
   repository. A repository without an `AGENTS.md` gets one. Each lists every
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
