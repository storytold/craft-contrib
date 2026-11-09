# RFC-0029: Using other companies' names

This RFC sets one rule for when the Craft apps use names that belong to other companies, so the answer is the same in every app and easy to check.

This is not legal advice.

## Rule

Don't use a name in the interface when either is true:

1. **It names another company, product or named technology.**
2. **Its owner claims it as a trademark for software.** That means it is on the owner's published trademark list, the owner marks it ™ or ® in its own documentation, or it is a known registration. A listing for unrelated goods doesn't count.

Use our own descriptive name instead. Common feature names that don't meet either test are fine.

A maintainer may also choose a different name for clarity, for example when a name doesn't describe our feature accurately or is inconsistent within the suite. The pull request says why.

## Names of things we work with

Use the accurate name, only where it refers to that thing:

- file formats, programming languages and APIs the app supports;
- data and names that come from other apps' files;
- protocols and plug-in hosts, once the integration works, following the owner's guidelines. Until then, hide the entry or show a disabled placeholder that doesn't claim the feature.

Factual compatibility statements ("opens PSD files") are fine in documentation, help and import messages. They don't belong in product names, taglines or menu labels. Don't use other companies' logos or icons.

## Applying it

- Change labels only. Command ids, preference keys, saved data and file-format strings stay the same.
- When a label changes, keep the previous label as a search alias.
- Record label changes in the app's `docs/feature-names.md` with the reason and a source link.
- Check the owner's list when a change is proposed; lists change over time.

## Out of scope

The apps' own names, interface design, icons and patents.

## House rules

This RFC doesn't change `standards/house-rules-core.md`: the block copied into each `AGENTS.md` is at its 3 KB limit. If the rule should reach agents through that block, a follow-up can propose a line once there is room.
