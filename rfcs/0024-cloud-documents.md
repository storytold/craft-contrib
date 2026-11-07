# RFC-0024: Cloud documents across the suite

Renumbered twice, apologies for the churn. The draft's 0001 was taken by #2 and #12. The interim 0018 collided with #30. 0024 is free.

A note on RFC-0009 (storytold/craft-contrib#10). That RFC is not accepted yet. This RFC does not depend on it. The draft asked for RFC-0009's exception. This revision states the criterion inline. This RFC therefore stands on its own whatever happens to #10.

## Summary

The suite gains remote Open, Save and Share through two pieces already built in `craft-libs`. The vendor-neutral `craft-cloud` contract is one. The `craft-ui` pages behind the `CloudBackend` / `ShareBackend` traits are the other. An app depends on these two things only. It never depends on a specific provider. It never depends on `collab-craft` or any other backend directly.

The first delivery is deliberately narrow. Remote Open and Save against a configured provider. Sharing where the provider supports it. Conflict handling with a clear refusal path (`Write` refuses to overwrite a newer version; the pages surface that as a conflict the user resolves). Ranged reads (`Files::read_range`, `stat`) are part of the contract, serving the web runtime's streaming needs (RFC-0023), with whole-read fallback where a backend refuses ranges. Live collaboration is a separate track with its own decision, detailed below.

Accounts and tokens stay out of this RFC. `craft-suite-account` and the launcher's keychain work (storytold/craft-launcher#366) wait for the shared keychain research already underway on the maintainers' side.

## Architectural boundaries

These boundaries respond to review on this PR. They bind every PR in the track.

**1. Configuration follows the craft-ai-integrations platform contract.** A person configures a storage connection once, in the launcher or in any individual app, through the same shared components, exactly as AI connections work under [craft-ai-integrations PLATFORM-CONTRACT](https://github.com/storytold/craft-ai-integrations/blob/main/docs/PLATFORM-CONTRACT.md). Every supporting app can then use that connection. The launcher stays optional. Secret storage for these connections lands with the keychain work, outside this RFC.

**2. Native apps connect directly to providers.** A native app speaks to Nextcloud, ownCloud or a plain WebDAV server through a provider adapter implementing `CloudBackend`. Remote storage never requires deploying CollabCraft or any other service. The direct-access pieces already exist as code. `craft-webdav` is a plain WebDAV client. `craft-nextcloud` speaks Nextcloud's WebDAV and OCS endpoints. Both currently live in the collab-craft tree. Under this RFC they move to the shared layer as the direct adapters, in their own small PRs after the contracts merge. A git-backed adapter (a repository as the storage target, already prototyped for the CAD apps' git-backed saves) joins the same list on the same terms. `craft-cloud-local`, a local-folder adapter in the contracts PR itself, is the reference implementation and the test vehicle.

**3. CollabCraft is one optional backend.** It remains a hosted deployment for the people who want one. Nothing in the shared layer names it or needs it.

**4. Web file handling is separate from the native configuration model.** A web build gets file access through its hosting gateway. The existing CollabCraft service can serve that role where it is deployed. The gateway's hosting and session requirements stay out of the native model. A native app's configuration never grows web-only concerns.

**5. Live collaboration is its own track.** Presence, relays and concurrent text are out of scope here. The pieces exist and are listed below so the work stays discoverable. Each waits for its own decision.

## Mapping the current implementation onto these boundaries

What stands unchanged. The `craft-cloud` contract (about 490 lines, serde and thiserror only, no vendor names) and the `craft-ui` cloud/share pages already match boundary 2's shape. The pages talk to a `CloudBackend`. Any adapter satisfies them.

What moves. `craft-webdav` and `craft-nextcloud` move out of the collab-craft tree into the shared layer, becoming the direct adapters boundary 2 requires. That is extraction, with no rewrite expected.

What changes. App integrations wire a configured connection (boundary 1) to an adapter (boundary 2). The CollabCraft client construction they carry today goes away. The pilot apps demonstrate that wiring.

What waits. `craft-suite-account` (keychain research). The web gateway's session model (boundary 4, its own later discussion). Everything in the live-collaboration track.

## The PRs in this track, in landing order

1. **storytold/craft-libs#160, cut down.** `craft-cloud`, the `craft-ui` cloud and share pages, their tests, plus `craft-cloud-local`. This is the first merge after acceptance. The criterion it meets, stated inline per the note above: it is a platform contract intended for every app, with wdunn001 as named maintainer.
2. **Direct provider adapters.** `craft-webdav` and `craft-nextcloud` as their own small PRs into the shared layer, after the contracts merge.
3. **craft-esign plus `craft_ui::esign`.** Its own draft PR, storytold/craft-libs#661, governed by RFC-0021.
4. **craft-tier-store, craft-worker-host, craft-runtime, the JS assets.** Its own draft PR, storytold/craft-libs#662, governed by RFC-0023.
5. **The craft-gpu family** (`craft-gpu`, `-wgpu`, `-null`, `-providers`). Its own draft PR, storytold/craft-libs#663, governed by RFC-0022.
6. **craft-suite-account.** Its own draft PR, storytold/craft-libs#665, governed by RFC-0025 (storytold/craft-contrib#35).
7. **storytold/craft-libs#345, craft-crdt.** Concurrent text without region locks. Part of the live-collaboration track, listed here so the track stays whole. It waits for the live-collaboration decision, separate from this RFC.
8. **Pilot app integrations.** A small number of representative apps (two, per review precedent) integrate after the library merges, to evaluate the interface. The remaining apps follow once the interface has settled.

The pilot docs and consumer screenshots currently in #160 move to the app PRs they describe.

## Conflict handling, as acceptance tests

The overwrite-refusal guarantee is an explicit acceptance test for every adapter, exercised by the contracts tests, by `craft-cloud-local`, by each provider adapter as it lands.

1. Two clients read the same document version. Both save. Exactly one succeeds. The other receives a conflict. No save ever silently overwrites the first.
2. The conflict path preserves both sides. The losing client keeps its unsaved edits. The newer remote version stays available. Nothing is discarded until the user chooses how to resolve.
3. Enforcement strength is a behavioral property of the provider and its deployment, never something the interface decides. The contract carries the precondition (`write` takes the etag the caller last saw) and the provider brings whatever enforcement it has. The two reference implementations already differ exactly this way. The local adapter checks the etag and then writes, a client-side check with a race window, honest best effort. The WebDAV path sends `If-Match` for the server to enforce atomically, where that server honors it. The acceptance tests are therefore the probe: run against a live provider, the double-save case reveals what the deployment actually guarantees. An adapter documents its enforcement point (client-side check, or server-side conditional) and never claims atomicity the test cannot demonstrate. The pages surface the weaker guarantee to the user where that is what the provider gives.

## Reference implementations, non-normative

Working code exists for every piece this RFC names, offered as reading aid and evidence, never as a required dependency. The contract and pages are in the cut-down storytold/craft-libs#160, with `craft-cloud-local` as the in-tree local-folder adapter. The direct provider clients are `crates/webdav` and `crates/nextcloud` in https://github.com/wdunn001/collab-craft, the extraction sources for boundary 2. The hosted-gateway shape of boundary 4 is that repository's relay service, running in one private deployment today. A reviewer can read any of these to see a boundary in practice. None of them is normative. The traits in the contracts PR are the only interface an app may depend on.

## What changes once this merges

storytold/craft-libs#160 gets cut down to item 1 and becomes reviewable on its own. The adapter extractions follow. The four app PRs closed earlier (storytold/designcraft#878, storytold/lightcraft#2060, storytold/effectcraft#1623, storytold/deckcraft#711) come back as new PRs against the merged library, in the pilot-first order above.

## Open questions this RFC leaves for later proposals

Where tokens live (the keychain RFC). The live-collaboration decision, covering craft-crdt, relays and presence. The web gateway's hosting and session model. Whether `collab-craft` transfers into storytold. That question matters only to the optional hosted backend. It no longer gates anything in this track.
