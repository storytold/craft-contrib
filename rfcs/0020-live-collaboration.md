# RFC-0020: Live collaboration: presence, invites, chat, concurrent text

RFC-0018 (storytold/craft-contrib#27) split live collaboration out of the cloud-documents track for its own decision. This is that proposal. It covers the four surfaces that need a live session: presence, one-time-key invite sharing, chat, concurrent text editing.

## Summary

The shared layer gains contracts for live sessions, with the same shape RFC-0018 set for storage. Apps depend on vendor-neutral contracts and shared UI only. Backends are swappable. No app names a service. Every surface degrades cleanly to absent when no live backend is connected, the same way the Open and Save pages hide without a storage backend.

## The four surfaces

**1. Presence.** Who else has this document open, shown in the shared presence control already in the contracts slice of craft-libs#160. The data model is ephemeral latest-wins state per peer and kind (cursor, selection, camera), handed to a joining peer and allowed without edit permission. Presence state is never persisted and never enters the document.

**2. One-time-key invite sharing.** A person mints an invite for a document. The invite carries a one-time key. Opening it grants the named access once, after which the key is dead. The flows exist in the prototype work: keyed invite minting bound to an app, invite arrival and open on both desktop and web, guest access with no account. The contract names mint, arrive and open. Key format, expiry and revocation live behind the trait.

**3. Chat.** The shared layer carries a conversation contract only: a document maps to a conversation, with message read and post as optional capabilities a backend may refuse. The chat user interface belongs to the host (a hosted shell, a provider integration, a native app), never to the shared editor UI. This boundary was learned the hard way: an in-editor chat panel was built, shipped and reverted in the prototype track because the editor must stay a bare, portable client. "Open chat" in shared UI opens the host's surface.

**4. Concurrent text.** `craft-crdt` (storytold/craft-libs#345) is the text layer: fractional keys, RGA text, replica merge, an in-crate server oracle, an N-client simulator with property tests. Structured commands keep relay sequencing and scope locks. Text takes the lock-free CRDT path. The two models share one protocol, proven in the prototype relay.

## Hard-won constraints the contracts must encode

These came out of the prototype at real cost. The contracts should make violating them impossible or loud.

1. A document is served by one relay. Fan-out does not cross instances. Session affinity binds to the document, never the client.
2. A joining peer receives sequence state reconciled with the persistent counter, never a fresh in-memory zero. The prototype's hardest bug was a relay restart desynchronizing the in-memory sequence from the persistent one.
3. Identity spaces for CRDT replicas are granted by the relay, never self-assigned. Two peers minting identical node identifiers silently drop each other's edits as duplicates.
4. Opaque frames pass through unparsed. A signed hash chain therefore survives the relay. Ephemeral state is latest-wins per peer and kind and requires no edit permission.
5. The room key is a stable document identifier, never a path. A path-keyed room splits on rename.

## What this does and does not decide

This RFC decides the contract shapes and their home (the shared layer, beside `craft-cloud`), the degrade-to-absent rule, the constraint list above. It does not decide which backend hosts relays, whether the suite runs a hosted service, or how this relates to the CraftHub (storytold/figcraft#357) and Macula mesh (storytold/wordcraft#4639) proposals. Those remain their own tracks. A mesh transport that satisfies the same contracts is a conforming backend. That is the point of the seam.

## Landing order

1. This RFC is accepted.
2. storytold/craft-libs#345 (craft-crdt) reviews on its own merits as the text layer.
3. A contracts PR adds the presence, invite and conversation traits beside `craft-cloud`, with an in-process mock and the degrade-to-absent tests.
4. Backends follow as adapters, each its own PR, none required by any app.

Related to storytold/craft-libs#123. Context: storytold/craft-contrib#27 (RFC-0018), storytold/craft-libs#345, tangency-labs/solidcraft#5184 (the upstream research thread the frame and ephemeral kinds answered).
