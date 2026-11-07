# RFC-0021: An e-signature contract for the suite

The esign slice was split out of storytold/craft-libs#160 by review, into its own draft PR: storytold/craft-libs#661. This RFC gives that slice the suite-level decision it needs before merging.

## Summary

The suite gains a vendor-neutral e-signature contract, `craft-esign`, modeled on the envelope lifecycle the industry's dominant provider interface defines. DocuSign's eSignature API is the reference semantics: an envelope holds a document snapshot, recipients with routing, field placements, a status that moves from draft through sent to completed, with void and remind as operations along the way. The contract reproduces that model in Rust with no vendor named anywhere. DocuSign becomes one adapter. Any CollabCraft deployment becomes another. A test double is a third.

## What exists

`crates/esign` is already written to this shape, held back from craft-libs#160's cut-down pending this decision. Its own documentation states the rule: "No type here names a vendor." The surface today:

1. `Envelopes` trait: `create` (from an `EnvelopeDraft` plus opaque document bytes, with an idempotency key so a retried create returns the same envelope), `send`, `status` (with per-recipient progress), `completed` (the finished document plus provider evidence, refused unless the status is Completed), plus defaulted-to-`Unsupported` operations for `void` and `remind`, the same optional-capability pattern `craft_cloud::Conversations` uses.
2. Types: `Envelope`, `EnvelopeDraft`, `EnvelopeStatus`, `Recipient`, `RecipientStatus`, `FieldPlacement`, `CompletedDocument`, `EnvelopeEvent`, `EsignCapability`.
3. A document is opaque bytes plus a content hash. The crate knows nothing of PDF structure. The legal audit trail, identity verification and invitation mail belong to the signing service.
4. It sits beside `craft-cloud` and reuses `craft_cloud::Identity`. It deliberately does not extend `Provider` or `Capability`. Signing is one thing. File access is another. A signer role is separate again from a share permission.

## The DocuSign mapping, as the audit for completeness

The contract's vocabulary maps onto DocuSign's as follows. Envelope maps to envelope. `Recipient` with routing maps to recipients with routing order. `FieldPlacement` maps to tabs. `EnvelopeStatus` maps to the envelope status lifecycle. `void` and `remind` map to the same-named operations. `completed` returning provider evidence maps to the completed-envelope documents plus the certificate of completion. `EnvelopeEvent` maps to Connect webhook events.

Known gaps against the reference interface, each needing a named triage decision on the PR: templates (reusable envelope definitions), embedded signing (a ceremony URL surfaced inside the app), bulk send, payment tabs. None block a first delivery. Each gap gets a named decision: add to the contract, declare an adapter-level extension, or document as out of scope.

## Reference implementations, non-normative

The contract itself is the held-back `crates/esign` in the craft-libs#160 branch history, readable today, with its in-process patterns mirrored from `craft_cloud::Conversations`. A hosted adapter was prototyped against a CollabCraft deployment in https://github.com/wdunn001/collab-craft, useful as a reading aid for how an adapter drives the ceremony. Neither is normative. The trait surface in the draft PR is the only interface an app may depend on, with DocuSign and any other provider equally behind it.

## What this RFC decides

The contract lives in the shared layer beside `craft-cloud`. It keeps vendor neutrality as a hard rule. It uses the DocuSign envelope model as its reference semantics, with the gap triage above. It ships with an in-process test double. Consumers: PdfCraft is the first named app. The second consumer or an argued exception lands on the draft PR per review precedent. Adapters (DocuSign, a CollabCraft deployment) follow as separate PRs, none required by any app, with the signing UI degrading to absent when no adapter is connected.

Related to storytold/craft-libs#123. Context: storytold/craft-contrib#27 (RFC-0018), storytold/craft-libs#160 (the review that split this out).
