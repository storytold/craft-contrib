# RFC-0023: A web runtime for the Craft apps

The runtime slice (tier-store, worker-host, runtime, the JS assets) was split out of storytold/craft-libs#160 by review, into its own draft PR: storytold/craft-libs#662. This RFC gives that slice the suite-level decision it needs before merging. It completes the RFC coverage of that PR: storage is RFC-0024, accounts RFC-0025, live collaboration RFC-0020, e-signature RFC-0021, the GPU layer RFC-0022.

## Summary

A `-web` build of a Craft app runs inside a browser tab with hard memory limits, an async-only persistence layer (OPFS, File System Access), plus work that belongs off the main thread. The web runtime composes three shared crates into one answer: `craft-tier-store` for tiered storage, `craft-worker-host` for the worker message protocol, `craft-runtime` to compose those with GPU dispatch (RFC-0022) behind one surface the app uses. The design goal throughout is that a large document works in a tab that cannot hold it whole.

## What exists

1. `craft-tier-store`. A synchronous mirror over asynchronous persistence, with a hot-tier LRU, key classes, a tier policy, versioned mirrors and pending-write tracking. The portable model lives in the crate. The JS host hooks (OPFS, File System Access) live in `craft-runtime`'s wasm glue. The model therefore stays testable natively.
2. `craft-worker-host`. The Rust side of the worker message protocol (`HostMessage`, `WorkerMessage`, pending flush entries), with the JS bootstrap as an asset.
3. `craft-runtime`. The composition crate: tier storage plus worker protocol plus GPU dispatch, re-exporting the pieces an app actually touches, with a `cloud-delegate` feature connecting tier storage to cloud blobs.
4. Ranged streaming. Cloud-backed keys hydrate and flush by byte range. A blob therefore never has to sit whole in the wasm heap. `craft_cloud::Files::read_range` and `stat` are implemented contract surface. Multipart upload is design-only and stays an open item below.

## Boundaries

**1. The tab holds no provider credentials.** A web runtime reaches storage through a `craft-cloud` HTTP gateway. Provider secrets, WebDAV details and account tokens stay server-side with the gateway (RFC-0024 boundary 4, RFC-0025). Nothing in these crates ever sees a provider credential.

**2. The portable model stays portable.** `craft-tier-store` compiles and tests natively. Browser specifics live only in the wasm glue. A native app may reuse the tier model where it helps, with no browser assumptions following it.

**3. GPU dispatch is RFC-0022's.** `craft-runtime` composes the GPU layer and adds no GPU semantics of its own. Capability questions route to `DeviceCaps` there.

**4. Ranged access is contract surface, never adapter magic.** `read_range` and `stat` live on `craft_cloud::Files`. Any backend can therefore serve them, with the runtime working against the contract alone. A backend without ranged support refuses honestly and the runtime falls back to whole reads where size permits.

## Acceptance requirements

Three requirement sets make the large-document and asynchronous-persistence claims verifiable, per review. The tiering model behind them is production-proven: this runtime serves one live deployment today. The same tier discipline (GPU buffers hot, wasm heap warm, OPFS or a chosen folder cold) runs large-model inference in a second production system, where the real ceilings were measured the hard way.

**1. "Flushed" is defined, in two distinct steps.** Acceptance into the hot mirror is step one. Successful persistence to the intended tier is step two. Only step two may mark a revision saved. Every pending write carries the revision it captured. A flush acknowledgment marks saved only the revision it actually persisted. A late acknowledgment for an older flush can neither overwrite nor mark saved a newer edit. A failed or partial flush restores the unaccepted writes to the pending queue for retry, with per-key results, never a bare count. Quota exhaustion, permission loss and worker restart are each a named test case. An acknowledged save survives closing and reopening the document.

**2. Memory is bounded by configured budget caps per tier, never by a claim about total memory.** A browser cannot know total memory, and the runtime never pretends to. Each tier gets a budget cap: the cold tier sized against what the platform reports (`navigator.storage.estimate()`), the warm tier against the wasm heap's practical ceiling, the hot mirror against its configured byte budget. The caps count everything: resident bytes, pending writes awaiting flush, retained evictions, in-flight ranges, message copies crossing the worker boundary. Exhausting a budget produces backpressure or an explicit refusal, never silent growth. The whole-read fallback checks `stat` before buffering and refuses a file past the budget, including when a server ignored a range request and returned everything.

**3. Worker ownership and isolation are stated.** The shared worker owns persistence for an app's documents, with namespaces separated by app and by document. Simultaneous tabs on one document go through that single owner. Every flush result is tied to the document revision it persisted. The dedicated-worker fallback declares its concurrent-writer policy explicitly: last-writer-wins per revision check, with the same revision-tied acknowledgments, never a silent weakening of the shared-worker guarantees.

**Pilot acceptance.** The named pilots (EffectCraft and FilmCraft) run real-browser tests covering a file larger than the configured memory budget, a slow and a failing persistence layer, and two tabs editing the same document concurrently. The gateway and session design stays with RFC-0024 and RFC-0025.

## Reference implementations, non-normative

The three crates and the JS assets are in the craft-libs#160 branch history, now its own draft PR. The ranged streaming design, with its v1 implementation notes, is the range and chunk API document in the same tree, prototyped against the gateway in https://github.com/wdunn001/collab-craft. None of this is normative. The crates' public surface in the draft PR is the only interface an app may depend on.

## What this RFC decides

The runtime lives in the shared repository as the three crates plus assets. The no-credentials-in-the-tab rule is a hard boundary. The portable model stays natively testable. Ranged reads are contract surface on `craft_cloud::Files`. Consumers: the `-web` builds of the apps, named concretely on the draft PR per review precedent.

## Open items

Multipart upload (design-only today; it lands as a contract capability a backend may refuse, the same optional pattern as everything else). The gateway's hosting and session model remains RFC-0024's deferred question and is decided there, never here.

Related to storytold/craft-libs#123. Context: storytold/craft-contrib#27 (RFC-0024), storytold/craft-contrib#38 (RFC-0022), storytold/craft-libs#160.
