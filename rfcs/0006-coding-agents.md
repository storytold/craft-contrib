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
7. **Keep PRs moving.** Concurrent PRs are welcome; address actionable feedback
   using the response expectations in core point 14 below.
8. **Be gentle with GitHub.** Reuse search results, avoid polling loops, and
   stop when you hit a rate limit instead of retrying in a loop.
9. **A human is responsible.** The person running the agent reads review
   comments and answers them, and is accountable for what the agent opens.

## Keeping the rules in sync

### Applicability

The core applies to apps and shared repositories. Select technical rules by
repository type rather than applying every Rust rule to every repository that
contains Rust. Repository-specific instructions may be stricter than applicable
shared rules. Any exception needs a maintainer-approved rationale in the local
instructions; adopting this RFC does not silently require an architectural rewrite.

In particular, allow a documented tooling exception for Craft PR Review's existing
Tauri/TypeScript UI: its [merged manifest](https://github.com/storytold/craft-pr-review-tool/blob/197a26dc9f03825bfb72b8e2cc376af87b54e4d8/crates/craft-pr-review/Cargo.toml#L21-L22)
uses Tauri. This exception does not relax the native Craft apps' Rust-only rule.

### Source and validation

Keep the canonical text in `standards/house-rules-core.md` in this repository,
with `standards/house-rules-rust.md` for Rust repositories. These are living
documents to be published during rollout: propose changes here for maintainer
review, rather than editing the copies. If RFC-0013 is accepted, use its point 5
for living documents.

Copy the applicable text into a generated block in each root `AGENTS.md`;
linking alone is not enough because agents may not follow links. For example:

```markdown
<!-- craft-house-rules:start core source=<full merged commit SHA> sha256=<content digest> -->
...generated text...
<!-- craft-house-rules:end -->
```

Record core and Rust source revisions and digests independently, using a separate
block for each. Define the same text and newline normalization for the generator
and validator. Checks in `craft-common-actions` validate each block against the
canonical file at its recorded revision and verify that revision belongs to the
merged source history. A digest edited alongside the text is not proof of integrity.
An edited block or invalid source reference fails the integrity check. An intact
older version only warns about freshness. Report inability to check the latest
source separately; do not describe it as tampering or as confirmed freshness.

### Updates

Prefer automatically opening update PRs, with maintainers reviewing and merging
them. A maintainer-run command is an acceptable starting point; automation is not
a prerequisite for adopting this RFC. Both approaches use only merged canonical
text, keep at most one open synchronization PR per repository, preserve local
instructions, and link the source revisions with a short summary of rule changes.

Repository-specific rules remain below the blocks, subject to the applicability
and documented exceptions above. They should cover exact commands, boundaries,
task-specific reading, and guards that look removable, with the reason for each.

Where `CLAUDE.md` exists, it should import the same instructions with
`@AGENTS.md`; rare Claude-specific rules can follow. An import avoids symlink
portability problems on Windows. Budget 3 KB for the core block and a 10 KB
soft limit for the root file, measured in bytes. Measure the pilot rather than
assuming that longer context improves results.

## Shared text

This is the text for the source files. It incorporates the nine points above
and rules repeated in existing repositories. References to other open RFCs
describe proposed policy, not already accepted policy.

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
8. **Changes reach `main` through a pull request.** Do not push directly.
   Branch protection enforces this. There is no maintainer exception.
9. **Stop and ask only about:** licensing outside the allowlist, publishing,
   releases, pushing to a new remote, secrets, spending money, destructive
   git, and anything else that is hard to undo. Otherwise keep working.
   `plan/` is not a source of rules. It is a gitignored maintainer notebook.
   An agent that does not have it does not ask for it and does not stop.
10. **Keep secrets and personal details out** of the repository, logs and
    outgoing requests.
11. **CI stays shared.** Workflows call `storytold/craft-common-actions`. Do not
    add or change CI in a feature or fix PR, even when related: CI changes get
    separate PRs for independent review. If CI cannot cover the change, link
    or open an issue there.
12. **Guards say why.** A check that looks removable carries a comment saying
    what it prevents. Read that reason before changing it.
13. **Keep instructions usable by everyone.** Do not add machine-specific notes
    or send readers to files they cannot open.
14. **Keep PRs moving.** Concurrent PRs are welcome; leaving actionable feedback
    unaddressed is the concern. Respond within 7 days, with a reminder or escalation
    at 14 days and eligibility for closure at 30 days. Count elapsed time from
    an identified actionable request while it awaits the author, not from PR
    creation. Optional suggestions and unconfirmed bot findings do not start the
    clock. A substantive fix, explanation or agreed plan addressing that request
    resets its clock if author action remains; unrelated activity does not.
    Pause while awaiting reviewer clarification or a maintainer decision. PRs
    waiting on maintainers do not count against contributors. Maintainers resolve
    disputed cases and decide closure; closure is never automatic. Reuse search
    results, avoid polling loops, and stop at rate limits.
15. **No agent attribution.** A pull request does not say that an agent wrote
    it.

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
4. **Rust only for native Craft apps.** No handwritten JavaScript or TypeScript,
   Tauri, Electron or webview shell in those apps. Shared tooling follows its
   documented technical scope and maintainer-approved exceptions, including the
   Craft PR Review exception above. Preserve any existing web build.
5. **Time goes through `chrono`.** Do not hand-roll calendar conversion; for
   dates beyond its range, preserve the existing whole-400-year Gregorian-cycle
   exception documented in [craft-libs](https://github.com/storytold/craft-libs/blob/main/AGENTS.md).
   Publish that guidance as `standards/time.md` before switching references to it.
6. **Contributor credits.** Add or edit only your own person's entry in
   `contributors/people.toml`, and only after asking them. Never guess a name or
   hand-edit `contributors.json`.

## Rollout

1. Publish the source files and checks, documenting technical applicability and
   approved exceptions before copying rules into each repository. If the
   licence allowlist still lives only in `craftrules`, copy it into these
   rules. Do not publish the rest of that repository.
2. Pilot on craft-ai-integrations, craft-libs and FigCraft, and measure the
   result and file sizes.
3. Move the remaining repositories in small PRs. Each PR lists every removed
   rule and where it went, so no guard disappears quietly. Delete links to
   `craftrules`.

## Alternatives

- **Separate rules per repository.** This is the current state; the same
  lessons have to be learned again in every repository.
- **Ban or limit AI-assisted PRs.** The craft apps chose to welcome them; this
  RFC keeps that choice and makes the volume easier to handle.

## Decided

- **`main` is pull requests only.** Branch protection is on. Nobody pushes to
  it directly, including maintainers.
- **`craftrules` is retired.** Its old tree is not published here. The house
  rules replace it. Rollout copies the licence allowlist across if these
  rules do not already contain it, then deletes the links. That is not
  another RFC.
- **`plan/` decides nothing.** The stop-and-ask list in the shared text is
  the only one. A missing `plan/` is not a reason to stop or to ask for the
  folder.
- **No agent attribution.** A pull request does not say that an agent wrote
  it. That line is redundant here, and a repository rule may not require it.
