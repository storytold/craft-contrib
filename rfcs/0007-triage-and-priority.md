# RFC-0007: Triage, labels and priority

Related: storytold/craft-libs#119, storytold/gridcraft#894.

## Summary

Write down what the triage labels mean, so they mean the same thing in every
craft app. Most labels already exist and are set by the TicketCraft triage
bot; this RFC defines them, adds a difficulty label and an "already fixed"
label, sets how priority is decided, and keeps closing issues a human
decision.

## Motivation

New issues arrive faster than anyone can read them. Without shared labels,
contributors pick issues at random, easy issues get several PRs while real
breakage waits, and issues that are already fixed, duplicates or design
questions eat time.

The TicketCraft bot already labels new tickets and issues, which helps. But
the labels can't yet tell a report from code reading alone apart from a
failure users actually hit: some code-audit findings with no reproduction are
marked critical, while data loss in a normal workflow or a broken core tool
sits at the same level or lower. People already build priority lists from
these labels, so they need a written meaning.

## Proposal

1. **Existing labels keep their names** and mean:
   - `duplicate`: the same problem as an earlier issue, which is linked;
   - `regression`: a problem that was fixed before and is back;
   - `needs info`: a real report that is missing what is needed to act
     (steps, expected vs. actual, version, OS, sample file);
   - `multiple issues`: the issue reports several separate problems and should
     be split. A single feature request described in several parts is one
     issue, not several;
   - `shared`: the problem affects several apps (a shared component or the
     installer);
   - `support`: not a development issue (how-to, installation, account);
   - `spam`, `no action needed`: as named.
2. **New labels**
   - `already fixed`: the problem no longer happens on `main`; the fixing
     commit or PR is linked.
   - **Difficulty**, one per bug: `trivial`, `easy`, `medium`, `hard`. "Easy"
     means a clear cause found in the code, a fix in one place, and a test.
     This lets newcomers find a first issue and lets experienced contributors
     leave easy ones to them. In TicketCraft this is a label group.
3. **Priority** (`priority: critical`, `high`, `medium`, `low`) weighs four
   things (suggested by @FredApps):
   - **Impact**: what goes wrong. Lost or corrupted work is worst, then a
     crash or hang, then a broken tool, then wrong output, then cosmetics.
   - **Reachability**: how a user gets there. A normal workflow counts most;
     an unusual file or setting less. A file someone else can send you
     (crafted PSD, SVG, PDF) still counts: a crash or memory blow-up on open
     can be a security problem.
   - **Evidence**: a reproduced failure counts more than a finding from
     reading code. Code-audit findings start one level lower until reproduced.
   - **Workaround**: if there is an easy one, one level lower.

   Levels, as a guide:
   - **critical**: lost or corrupted work, or a crash or hang, in a normal
     workflow; a regression in a core tool; a reproduced crash from a file a
     user can be sent;
   - **high**: a core tool broken or unusable; corruption reachable only with
     unusual files; credible but unreproduced resource-exhaustion findings;
   - **medium**: works with a workaround, layout problems, single-language bugs;
   - **low**: cosmetic, internal or test-only issues.

   Before ranking an issue, check whether part of it is already fixed on
   `main`, and rank only what is left. In TicketCraft this text goes into
   "What each priority means".
4. **Before labelling a bug**, check recent commits on `main` and search for
   duplicates; label `already fixed` or `duplicate` with a link instead.
5. **Issues with several problems** are split, one problem per issue, so each
   can be claimed and closed on its own.
6. **The bot labels, people close.** Automatic labels and comments are fine:
   a wrong label costs little and anyone can point it out in a comment. Actions
   that close an issue (`support`, `no action needed`) are only suggested by
   the bot and applied by a maintainer, because a wrong close leaves the
   reporter without an answer.

## Alternatives

- **Priority only from maintainers.** Accurate, but maintainers are the
  bottleneck already (RFC-0008).
- **No difficulty label.** Then every contributor re-reads the same issues to
  find something they can do.
- **Let the bot close issues automatically.** Saves maintainer time, but a
  wrong close is hard to notice and the reporter may not come back.

## Open questions

- Should `already fixed` close the issue once the fix is released, or right
  away with a link to the fixing PR?
- Exact colours: reuse the scheme in storytold/craft-libs#119?
