# RFC-0006: A shared section for coding agents (`AGENTS.md`)

Related: storytold/craft-libs#37, storytold/craft-libs#38.

## Summary

Add one shared section to every craft app's `AGENTS.md` with the rules that
apply to all of them: claim before working, verify instead of guessing, keep
fixes narrow, review before opening a PR, and don't flood the repositories.
Shared repositories are included as well as apps. Each repository keeps its
own rules below the shared text.

## Motivation

The craft apps welcome AI-assisted contributions, and much of the current PR
volume comes from coding agents. An agent follows its instructions literally,
so gaps in the instructions turn into the same mistake repeated many times:
duplicate PRs, fixes for bugs already fixed on `main`, PRs left as drafts,
or code that uses functions that don't exist.

Running agents on hundreds of issues across eleven repositories taught us
which instructions prevent those mistakes. This RFC proposes them as the
shared part.

## Proposal

The shared section says:

1. **Claim and check** as in RFC-0003: right before each issue, read its
   comments, search PRs for `#N` and keywords, and check recent commits on
   `main`. Claim on the issue before writing code. Release claims you won't
   finish.
2. **Verify, don't guess.** Before using a function, type or field, open its
   definition and use it exactly as defined. If you cannot build or run the
   tests locally, say so to the human running you, and rely on the project's
   CI before marking the PR ready.
3. **Notes from triage can be wrong.** A short note on where the bug is (from a
   person, a bot, or another agent) is a starting point. Confirm the cause in
   the code before fixing it.
4. **Narrow fixes and real tests**, as in RFC-0005.
5. **A second pass before opening the PR.** Review the diff against the issue:
   every identifier exists as used, the test fails without the fix, nothing
   outside the bug changed, and the description is true.
6. **Look for a better PR** before opening yours, as in RFC-0004, and close
   yours when the other is better.
7. **Don't flood.** Open PRs at a pace the maintainers can review, and don't
   open new ones while earlier ones have unanswered review comments.
8. **Be gentle with GitHub.** Reuse search results, avoid polling loops, and
   stop when you hit a rate limit instead of retrying in a loop.
9. **A human is responsible.** The person running the agent reads review
   comments and answers them, and is accountable for what the agent opens.

## Keeping the rules in sync

Keep the canonical text in `standards/house-rules-core.md` in this repository,
with `standards/house-rules-rust.md` for Rust repositories. These are living
documents: propose changes here for maintainer review, rather than editing the
copies. If RFC-0013 is accepted, use its point 5 for living documents.

Copy the applicable text into a generated block in each root `AGENTS.md`;
linking alone is not enough because agents may not follow links. For example:

```markdown
<!-- craft-house-rules:start core@<first 12 hex characters of the SHA-256> -->
...generated text...
<!-- craft-house-rules:end -->
```

The digest identifies the source text used to generate the block. Checks in
`craft-common-actions` should fail CI if a generated block has been edited,
but only warn when an intact block is behind the current source. A maintainer
runs a command to open a small update PR per repository; there is no scheduled
bot. Repository-specific rules remain below the block and may be stricter, but
not weaker. They should cover exact commands, boundaries, task-specific reading,
and guards that look removable, with the reason for each guard.

Where `CLAUDE.md` exists, it should import the same instructions with
`@AGENTS.md`; rare Claude-specific rules can follow. An import avoids symlink
portability problems on Windows. Budget 3 KB for the core block and a 10 KB
soft limit for the root file, measured in bytes. Measure the pilot rather than
assuming that longer context improves results.

## Candidate shared text

The following is proposed text for the source files, subject to the open
questions below. It incorporates the nine points above and rules repeated in
existing repositories. References to other open RFCs describe proposed policy,
not already accepted policy.

### Core

1. **A person is accountable.** The person running an agent answers for what it
   opens and answers its review comments.
2. **Check, then claim.** Right before starting an issue, read its comments,
   search PRs for `#N` and keywords, and check recent commits on `main`.
   Comment on the issue before writing code. Release a claim you will not finish.
3. **Verify, don't guess.** Open a function, type or field's definition before
   using it. Confirm the cause in the code, even when someone suggests a location.
4. **Keep fixes narrow and tested.** Fix what the issue describes, where the bug
   is. Add a test that fails without the fix.
5. **Review your diff before opening a PR.** Check identifiers, the test, scope,
   and the accuracy of the description.
6. **Look for a better PR.** Compare another PR that fixes the same thing; close
   yours if the other is better or equal, and say why.
7. **Report only what ran.** Say what you tested and what you could not run. If
   you cannot build or test locally, say so and rely on CI before marking ready.
8. **Changes reach `main` through a pull request.** Do not push directly. The
   possible maintainer exception is an open question.
9. **Stop and ask about decisions outside your remit:** licensing, publishing,
   releases, secrets and anything hard to undo. Otherwise keep working.
10. **Keep secrets and personal details out** of the repository, logs and
    outgoing requests.
11. **CI stays shared.** Workflows call `storytold/craft-common-actions`. Do not
    add or change CI in a feature or fix PR; if CI cannot cover the change, link
    or open an issue there.
12. **Guards say why.** A check that looks removable carries a comment saying
    what it prevents. Read that reason before changing it.
13. **Keep instructions usable by everyone.** Do not add machine-specific notes
    or send readers to files they cannot open.
14. **Go at a pace people can review.** Answer open review comments before
    opening more PRs. Reuse search results, avoid polling loops, and stop at
    rate limits.

### Rust

1. **Never crash.** Non-test code does not panic or use `unsafe`, except where
   the repository file names an exception. Return errors, treat input as hostile,
   bound recursion and allocations, and copy the `deny` attribute from an
   existing crate for new production crate roots. Test each crash fix with a case
   that panicked before the fix.
2. **Clean room.** Learn from observed behaviour and public specifications; do
   not copy another product's code, assets, strings or artwork. Check dependency
   licences before adding them.
3. **Assets and fonts.** Assets must be original or suitably licensed and
   recorded in `ATTRIBUTION.md` in the same change. Font files belong in
   `storytold/craft-fonts`.
4. **Rust only.** No handwritten JavaScript or TypeScript, Tauri, Electron or
   webview shell. Preserve any existing web build.
5. **Time goes through `chrono`.** Do not hand-roll calendar conversion; for
   dates beyond its range see `standards/time.md` in craft-contrib.
6. **Contributor credits.** Add or edit only your own person's entry in
   `contributors/people.toml`, and only after asking them. Never guess a name or
   hand-edit `contributors.json`.

## Rollout

1. Publish the source files and checks.
2. Pilot on craft-ai-integrations, craft-libs and FigCraft, and measure the
   result and file sizes.
3. Move the remaining repositories in small PRs. Each PR lists every removed
   rule and where it went, so no guard disappears quietly.

## Alternatives

- **Separate rules per repository.** This is the current state; the same
  lessons have to be learned again in every repository.
- **Ban or limit AI-assisted PRs.** The craft apps chose to welcome them; this
  RFC keeps that choice and makes the volume easier to handle.

## Open questions

- Should maintainers have an exception to the proposed PR-only rule for `main`?
  Some repositories currently permit direct pushes and others prohibit them.
- Should useful standards from `craftrules` be published here, or should that
  repository be retired? Some existing links are inaccessible to outsiders.
- Does the public list of decisions requiring a pause cover what currently lives
  in gitignored `plan/` files?
- Should agents mention in the PR that they were used? Existing repositories
  differ; should the core block give one answer?
- Should the shared rules cover shared repositories as well as apps, as proposed?
