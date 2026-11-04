# RFC-0001: Cloud documents across the suite

## Summary

The suite gains cloud Open, Save and Share through two pieces already built in `craft-libs`. The vendor-neutral `craft-cloud` contract is one. The `craft-ui` pages behind the `CloudBackend` / `ShareBackend` traits are the other. An app depends on these two things only. It never depends on a specific provider. It never depends on `collab-craft` or any other backend directly.

`collab-craft` becomes `storytold/collab-craft`. It is one reference `CloudBackend` implementation, talking to a Nextcloud or ownCloud deployment over WebDAV, CardDAV and CalDAV. It is not the only implementation. A second reference implementation, `craft-cloud-local`, ships alongside the contract in `craft-libs` itself. It proves the same point. Any backend that speaks the trait drops in unchanged.

Accounts and tokens are the most security-sensitive part of this track. `craft-suite-account` and the launcher's keychain work (storytold/craft-launcher#366) are deliberately left out of this RFC. They get their own proposal once this one settles the base contract.

## Motivation

Thirteen apps each want cloud Open, Save and Share. Several already have PRs open doing it against a personal fork. The shared contract is not yet in `storytold/craft-libs`. The reference backend is not yet in `storytold/collab-craft`. Reviewers are correctly declining those PRs regardless. A build that fetches a personal repository is not reviewable by storytold. This RFC is the suite-wide decision that unblocks them, cut down from storytold/craft-libs#160 per its review.

## What this RFC decides

**1. The contract.** The suite offers cloud Open, Save and Share. The only thing an app depends on is `craft-cloud` (names WebDAV, CardDAV, CalDAV, no vendor) and the `craft-ui` pages behind `CloudBackend` / `ShareBackend` (`cloud/`, `share/`, `account.rs`, `presence.rs`, `menu.rs`, `buttons::chevron`). An app writes `impl CloudApp` and supplies a backend. It never names a provider.

**2. `collab-craft`.** Transfers to `storytold/collab-craft` as the reference `CloudBackend` for Nextcloud and ownCloud. Maintainer is wdunn001. That covers releases, security fixes, the relay's deployment. It holds until the project names someone else or a team.

**3. Accounts and tokens.** Out of scope here. `craft-suite-account` and the launcher's keychain work need their own RFC before anything in that area merges. A suite-wide session store is too security-sensitive to settle as a clause in this one.

**4. Relationship to CraftHub and the Macula mesh.** This RFC does not choose between them. Each answers a different question. This RFC answers where a document's bytes live between sessions. It also answers how an app opens, saves and shares that document. CraftHub (storytold/figcraft#357) answers accounts, teams, permissions, review, comments, a marketplace. It is a collaboration and organization layer. It can sit above cloud-stored documents, local documents, or both. The Macula mesh (storytold/wordcraft#4639) answers two people co-editing live over a mesh, with no server and no account. Cloud storage keeps a server in the loop by design. The two are not substitutes. A document stored through `craft-cloud` can be the same document two people co-edit live over the mesh. CraftHub's accounts can sit above either. This RFC does not require the suite to pick one collaboration architecture.

**5. Landing order, per RFC-0009.** `craft-cloud` and the `craft-ui` cloud/share pages are a platform contract meant for every app from the start. Request RFC-0009's documented exception on that basis, with wdunn001 named maintainer. Everything else in storytold/craft-libs#160 lands as its own PR under RFC-0009's default rule, naming its consuming apps:

- `craft-esign` plus `craft_ui::esign`. Needs two named consuming apps, PdfCraft plus one more, or an exception argument.
- `craft-suite-account`. Waits for its own RFC (point 3).
- `craft-tier-store`, `craft-worker-host`, `craft-runtime`, the JS assets. Needs named consuming apps.
- `craft-gpu`, `-wgpu`, `-null`, `-providers`. Needs named consuming apps.
- The pilot docs and screenshots belong with the app PRs they describe.

## What changes once this merges

storytold/craft-libs#160 gets cut down. It keeps `craft-cloud`, the `craft-ui` cloud/share pages, their tests. That cut-down PR becomes mergeable. `wdunn001/collab-craft` transfers to `storytold/collab-craft`. App PRs currently pinning `wdunn001/craft-libs` or `wdunn001/collab-craft` retarget to the real storytold locations. The four closed PRs (storytold/designcraft#878, storytold/lightcraft#2060, storytold/effectcraft#1623, storytold/deckcraft#711) come back as new PRs at that point.

## Open questions this RFC leaves for later proposals

The account and token RFC is one (point 3). Each remaining crate split out of #160 is another, each on its own PR with named consumers and a maintainer (point 5). Whether CraftHub or the Macula mesh proceed is a third. Each proceeds on its own track, independent of this one.
