# RFC-0040: A shared list of checked names, and names in GitHub text

Updates: [RFC-0029](0029-other-companies-names.md)

This RFC gives the suite one list of names checked under RFC-0029, so a name
is checked once rather than in every app. It also says how the rule applies to
issue, pull request and commit text.

## Why

RFC-0029 has each app record its checks in its own `docs/feature-names.md`.
Apps that aim at parity with the same reference applications meet the same
names, and a name "already checked and used the same way" doesn't need a new
check. Neither helps unless an app can find the other apps' checks. Without a
shared record each app repeats the search and may decide differently.

PhotoCraft listed the reference-application names it expects to meet for
parity on [#43](https://github.com/storytold/craft-contrib/pull/43#issuecomment-6849272731),
and asked whether issue and pull request text is in scope. The approving review
left both as follow-ups that didn't block the rule. This RFC takes them up.

## Decision

### 1. One shared list

[`standards/feature-names.md`](../standards/feature-names.md) is the living
document ([RFC-0013](0013-amending-rfcs.md), point 5) for names checked under
RFC-0029. Each row gives the name, the decision, why, our name where an app has
chosen one, and a link to the check record.

- **Look it up first.** Before checking a name, look for it in the list. A row
  with a decision and a check record counts as "already checked" for any app
  that uses the name the same way.
- **Add the row.** An app pull request that records a check, or that drops a
  name under test 1, is followed by a pull request here that adds or updates
  the row and links the app's record. The app's `docs/feature-names.md` stays
  the full record; the list points to it.
- **Reuse our name.** When two apps have the same feature, the second one uses
  the name the first one chose, unless its pull request says why not. That
  keeps the suite consistent.
- **Rows are ordinary pull requests.** A row applies the rule; it doesn't
  change it. It needs a maintainer's approval and no comment period. A row that
  disagrees with an earlier decision goes to an owner, as RFC-0029's unclear
  cases do.
- **"Not checked yet" is a work list, not a finding.** Apps may list names
  they expect to meet, so others can see the work ahead. Listing a name there
  says nothing about who owns it.

### 2. Families of names

When one company uses the same word for a set of features (for example
`Smart …` or `Magic …`), the family can be checked once. The row names the
members it covers. A member with its own application or registration gets its
own row, and its own check result wins over the family's.

### 3. Names in GitHub text

Issue and pull request titles and bodies, review comments, commit messages and
discussion are working text. RFC-0029's rule for our features' names applies
to them as far as this:

- **Our feature goes by our name** once the app has chosen one, in titles,
  bodies and commit messages, so searches find it and the names don't drift
  apart.
- **Describing another product for comparison is allowed** there, as in
  RFC-0029's internal comparison documents. A repository may be stricter. For
  example, PhotoCraft writes "the reference application" and describes other
  features by what they do.
- **Existing text isn't rewritten for this RFC.** Update a title or body when
  you edit it anyway, for example when an app renames a feature that the issue
  is about.

Release notes, store text and READMEs are product text and already follow
RFC-0029 in full.

## Not in this RFC

- New tests or changes to the check itself. RFC-0029 still sets them.
- Legal clearance. The list records policy decisions, as RFC-0029's examples
  do.
- A line in `standards/house-rules-core.md`. The block is still at its size
  limit.

This RFC updates RFC-0029, so its pull request adds `Updated by: RFC-0040`
under RFC-0029's title.
