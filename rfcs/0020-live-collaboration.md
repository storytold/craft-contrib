# RFC-0020: Live collaboration: presence, invites, chat, concurrent text

RFC-0024 (storytold/craft-contrib#27) split live collaboration out of the cloud-documents track for its own decision. This is that proposal. It covers the four surfaces that need a live session: presence, one-time-key invite sharing, chat, concurrent text editing.

## Summary

The shared layer gains contracts for live sessions, with the same shape RFC-0024 set for storage. Apps depend on vendor-neutral contracts and shared UI only. Backends are swappable. No app names a service. Every surface degrades cleanly to absent when no live backend is connected, the same way the Open and Save pages hide without a storage backend.

## The four surfaces

**1. Presence.** Who else has this document open, shown in the shared presence control already in the contracts slice of craft-libs#160. The data model is ephemeral latest-wins state per peer and kind (cursor, selection, camera), handed to a joining peer and allowed without edit permission. Presence state is never persisted and never enters the document.

**2. One-time-key invite sharing.** A person mints an invite for a document. The invite carries a one-time key. Opening it grants the named access once, after which the key is dead. The flows exist in the prototype work: keyed invite minting bound to an app, invite arrival and open on both desktop and web, guest access with no account. The contract names mint, arrive and open. Key format, expiry and revocation live behind the trait.

**3. Chat.** The shared layer carries a conversation contract only: a document maps to a conversation, with message read and post as optional capabilities a backend may refuse. The chat user interface belongs to the host, never to the shared editor UI. This boundary was learned the hard way: an in-editor chat panel was built, shipped and reverted in the prototype track because the editor must stay a bare, portable client. "Open chat" in shared UI opens the host's surface.

The host surface takes two concrete forms. Both are options on web and on native alike. The deployment chooses, never the platform.

The external form puts chat, sharing and activity in a surface beside the editor, outside it. On the web that is a hosting page wrapping the embedded editor with a collaboration aside, the pattern Nextcloud uses with Collabora's web office: the office canvas is embedded, the aside is the host page's own UI. On native the same form is a provider's companion surface opened beside the app. The web aside requires a chrome-free embedded build of the editor and a host bridge for the aside to drive. The contracts PR must name both as deliverables.

The internal form puts the chat surface inside the app's own shell, fed by the same conversation contract. A native app uses it when no external surface serves the deployment. A standalone web build, with no wrapping page, uses it the same way. Shared optional widgets may exist for that surface, but the app owns placement, consistent with how the platform contract treats shared components everywhere else. The internal surface belongs to the app shell, never to the shared editor UI. The hard rule above therefore survives in both forms.

Both forms are capability-gated on the backend's answer and degrade to absent when chat is refused or no backend is connected.

**4. Concurrent text.** The `Texts` API is the contract boundary. The engine behind it is replaceable. Review has made the engine choice concrete. Testing against `craft-crdt` (storytold/craft-libs#345) found sibling ordering keyed on replica id with no logical clock, delete-before-insert resurrection, compaction divergence. Convergence tests passed because every replica agreed on the wrong answer. The suite therefore takes the library route as the default direction. A mature CRDT engine (yrs, loro or diamond-types) sits behind the existing `Texts` API, removing this defect class by design. `craft-crdt` may continue as an engine only if it passes a differential oracle suite against the chosen library (random concurrent edit traces, identical final states required). That oracle suite is a deliverable of the contracts PR either way. Structured commands keep relay sequencing and scope locks. Text takes the lock-free path. The two models share one protocol.

## Hard-won constraints the contracts must encode

These came out of the prototype at real cost. The contracts should make violating them impossible or loud.

1. A document is served by one relay. Fan-out does not cross instances. Session affinity binds to the document, never the client.
2. A joining peer receives sequence state reconciled with the persistent counter, never a fresh in-memory zero. The prototype's hardest bug was a relay restart desynchronizing the in-memory sequence from the persistent one.
3. Identity spaces for CRDT replicas are granted by the relay, never self-assigned. Two peers minting identical node identifiers silently drop each other's edits as duplicates.
4. Opaque frames pass through unparsed. A signed hash chain therefore survives the relay. Ephemeral state is latest-wins per peer and kind and requires no edit permission.
5. The room key is a stable document identifier, never a path. A path-keyed room splits on rename.

## Reference implementations, non-normative

The surfaces and constraints above came out of working code in https://github.com/wdunn001/collab-craft, offered as reading aid and evidence, never as a required dependency. Review against that prototype found real defects (the text-engine ordering class above, the reopen-after-idle sequence case, the reconnect lock case, the snapshot replay pairing), now filed on their repositories and named as conformance cases here. The prototype is evidence of the design and a source of its hard constraints. It is not yet a conforming backend. The conformance suite exists precisely to make that distinction mechanical. The two-model session (ordered submission, opaque scope locks, lock-free `crdt.*` passthrough) is `crates/collab`, with the lock-skip rule enforced in its protocol and pinned by its own tests. The relay half implements the constraint list: TTL'd atomic scope locks with refresh, sequence reconciliation on join, relay-granted identity spaces, opaque frame passthrough, document-id room keys. The ephemeral presence kinds and the keyed-invite flows are in the same tree. The text layer is `craft-crdt` at storytold/craft-libs#345, with its server oracle and N-client simulator. A reviewer can trace any constraint to the bug that forced it. None of this is normative. The contracts PR's traits are the only interface an app may depend on. A mesh or any other transport satisfying them is equally conforming.

## Conformance guarantees

Two sets of observable guarantees bind every conforming backend, with conformance tests in the contracts PR. The mechanisms stay behind the traits. The behavior is shared.

**Mixed editing and recovery.**

1. A structural command that deletes or moves an object while another peer edits its text has one defined outcome. The structural command wins visibility. Text operations arriving for a scope removed by an acknowledged structural command are dropped from the document deterministically, identically on every replica. Deleted text never resurrects the object. The drop is never silent for the person who typed: the originating client keeps a local recoverable copy of its orphaned text and surfaces it to its user. Someone's typing never just disappears. The prototype already holds ops for an unseeded scope and retains per-scope history. That is the shape this rule generalizes.
2. An operation is durable once the relay has sequenced it against the persistent counter. An acknowledgment means exactly that. The counter outlives the process (constraint 2 above). Nothing acknowledged is therefore lost by a relay restart.
3. Replay never loses an operation and never applies one twice. Sequenced operations dedupe by sequence number. CRDT operations dedupe by operation identity. The CRDT's own merge makes that idempotent. Reconnect, replay after restart, joining from a snapshot paired with its sequence state: each is a conformance case. A saved snapshot carries the sequence state of the operations it already contains. Replay applies only operations after it. The double-apply case (a saved document replaying ops already baked into it) is a named conformance test. Two further named cases come from review against the live prototype. Reopening a document after every peer has left reads the join sequence from the persistent counter, never from a fresh in-memory room, with an edit after reopen confirmed for its author and applied for the next joiner. A same-peer reconnect preserves the peer's identity and its locks, keyed on the session (never the socket), with the old socket's close unable to evict the live peer or hand its lock away.

**Invite redemption.**

1. Redemption consumes the one-time key atomically. Concurrent redemptions resolve to exactly one winner.
2. A redeemer whose successful redemption response was lost can retry safely. Redemption is idempotent for the same redeemer: the retry returns the same grant, never a second one.
3. Fetching or previewing the link never consumes the key. Consumption happens only on the explicit redeem call.
4. An expired key refuses redemption with a distinct, stated outcome. Revoking an unredeemed invite kills its key. Revoking an invite does not end access already granted through it. Ending granted access is a separate, explicit operation on the grant. The two acts are never conflated.

## What this does and does not decide

This RFC decides the contract shapes and their home (the shared layer, beside `craft-cloud`), the degrade-to-absent rule, the constraint list above. It does not decide which backend hosts relays, whether the suite runs a hosted service, or how this relates to the CraftHub (storytold/figcraft#357) and Macula mesh (storytold/wordcraft#4639) proposals. Those remain their own tracks. A mesh transport that satisfies the same contracts is a conforming backend. That is the point of the seam.

FigCraft's shipping collaboration (its collab crates, collab-net and figcraft-server, state-based last-writer-wins) relates the same way. The relationship has history in this track (storytold/craft-libs#123, the CraftHub brief). CraftHub (storytold/figcraft#357) is itself a generalization of figcraft-collab by its own description. That is why RFC-0024 treats it as the organization layer. FigCraft's server and its state-based model can implement these contracts for the surfaces they cover, with the structured-command path mapping naturally onto last-writer-wins state. Nothing here replaces FigCraft's in-app collaboration. Nothing here requires it to change.

## Landing order

1. This RFC is accepted.
2. A contracts PR adds the live-session contract and the presence, invite and conversation traits beside `craft-cloud`, with an in-process mock, the degrade-to-absent tests and the differential text oracle from the text-engine section. The contracts work is independent of accepting any particular engine. The session contract is the proven two-model protocol: ordered submission with opaque scope locks for commands that do not commute, plus lock-free `crdt.*` passthrough for text. A lock scope is an opaque string each app reads its own way. The contract therefore names no app and no provider. The lock-or-CRDT choice is per command kind, made by the app, carried by the protocol. Any backend implementing the contract supports both models.
3. storytold/craft-libs#345 (craft-crdt) is conditional on that oracle. The mature-library route stays the default per the text-engine section. #345 proceeds to review only if craft-crdt passes the differential oracle against the chosen library; otherwise it closes in favor of the library behind the same `Texts` API. The reference section continues to describe what the existing prototype uses, non-normative as marked.
4. Backends follow as adapters, each its own PR, none required by any app.

Related to storytold/craft-libs#123. Context: storytold/craft-contrib#27 (RFC-0024), storytold/craft-libs#345, tangency-labs/solidcraft#5184 (the upstream research thread the frame and ephemeral kinds answered).
