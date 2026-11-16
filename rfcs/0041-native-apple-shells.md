# RFC-0041: A scoped native Apple shell around a Rust engine

Status: Proposed; RFC number provisional until maintainers confirm it.

Updates: [RFC-0006](0006-coding-agents.md).

Related: [RFC-0039 / storytold/craft-contrib#60](https://github.com/storytold/craft-contrib/pull/60),
[RFC-0040 / storytold/craft-contrib#62](https://github.com/storytold/craft-contrib/pull/62),
[RFC-0036 / storytold/craft-contrib#56](https://github.com/storytold/craft-contrib/pull/56).

## Summary

Permit an individually approved, thin Swift/SwiftUI/UIKit iPhone/iPad host while
keeping one Rust engine, document model, renderer, command system and undo history.
This is the missing language-policy decision, not a replacement mobile rollout,
an accepted exception, or a requirement to migrate every app to a new framework.
The [native Apple boundary](../standards/apple-native.md) is the proposed living
contract. The same PR proposes the corresponding narrow canonical Rust-rule change;
no generated app instruction block is changed.

## Why a separate decision is necessary

The current native-app rule is Rust-only. A working fork is not permission to
silently weaken it. RFC-0039 currently retains that rule and proposes egui Mobile
Mode, Android first. Its author should incorporate or reject this alternative
explicitly; this PR does not edit another contributor's pending RFC branch.

There are now two materially different sources of evidence:

- [PdfCraft's upstream POC proposal](https://github.com/storytold/pdfcraft/pull/7549)
  uses a SwiftUI document workspace over existing Rust automation and page rasters.
  It does not host egui or consume `craft-mobile`. Its working mobile subset is not
  full desktop feature parity.
- [storytold/craft-libs#632](https://github.com/storytold/craft-libs/pull/632)
  proposes a shared egui/Metal host with native platform shells.
  Its author's [RFC discussion](https://github.com/storytold/craft-contrib/pull/60)
  additionally reports local, unpushed Rust-described native chrome. That later kit
  has not been independently reviewed here. Do not copy or prescribe an unavailable
  API, or describe one POC as validation of the other.

The host author reports an iOS scene-lifecycle limitation in winit and an iOS
accessibility-adapter gap. Treat these as version-specific findings to reproduce,
not permanent claims about every future winit or AccessKit version.

## Proposed decision

1. A maintainer can approve a documented iOS/iPadOS host exception for a named app,
   binding generator and revision. Desktop and web behavior remain unchanged.
2. Swift owns OS presentation, accessibility and platform resource lifetime only.
   Rust retains document authorization, transformations, validation, history and
   serialization. No PDFKit replacement engine, JavaScript/TypeScript, webview,
   Tauri or Electron is introduced by this exception.
3. Use generated, version-pinned bindings and typed bounded messages. Any necessary
   generated FFI `unsafe` must have a separate, narrowly documented audited boundary;
   this does not relax the general no-unsafe/no-panic rule.
4. Compare the existing host proposal before extracting common code. First use
   existing shared libraries. A new component needs independently exercised app
   adapters and behavior-preserving tests, not merely two copied UI files.
5. An optional Rust-described panel kit can be evaluated when its source is
   available. It must preserve command IDs and begin/change/commit/cancel/undo
   semantics. This RFC neither invents its API nor adopts it sight unseen.
6. Require universal device families and separate iPhone/iPad validation, including
   compact/split layouts, large text, file-provider lifetime and unsaved-work safety.
   Simulator tests are not physical-device, VoiceOver speech or release certification.
7. Unsupported desktop tools stay hidden and are tracked publicly with evidence.
   A feature-reduced beta must say so; it cannot satisfy another RFC's full-parity
   Stable criterion merely by changing its wording.
8. Resolve native-resource and screenshot policy explicitly before adoption.
   UIKit/SwiftUI uses OS-provided fonts and symbols by reference; these are not
   open-licensed files that an app may copy into its assets. PdfCraft's stricter
   asset policy currently also excludes their appearance in published captures.
   This RFC requests an app-owner-reviewed, narrow exception for SDK-authorized
   runtime chrome and genuine captures of that chrome, with provenance and the
   platform terms distinguished from contributor-original content. It does not
   authorize redistribution or relicensing of font/symbol files, third-party
   screenshots, or proprietary document content. Until the app's own policy is
   explicitly updated, publish only permitted document-canvas crops; keep full
   native-chrome captures private. Accepting a language exception alone does not
   override a stricter asset rule.

## User experience and maintenance impact

Keep desktop task order, terminology and safe mutations, not mouse-sized geometry.
The same tool intent may use an iPad sidebar and an iPhone sheet. Native Files,
sharing, text selection and accessibility replace custom OS imitations. Retain
desktop artwork/compatible metadata by reference or generation. System symbols
and a small native presentation adapter need not drag egui into a domain engine.

This adds native UI/SDK maintenance and two device-family gates; it does not make
mobile free to maintain. New shared native chrome should remove repeated adapters
only after it preserves existing behavior and reduces measured maintenance work.
Retaining native accessibility typography and standard OS controls avoids a
second custom UI toolkit, but their publication/licensing boundary needs the
explicit decision above. Replacing the whole OS chrome merely to obtain a
misleadingly open-licensed screenshot is not the proposed solution.

## Rollout and acceptance

- Maintainers reconcile this with RFC-0039, the host author and affected app owners.
- Follow RFC-0000's comment period and independent maintainer approval. User approval
  of a fork experiment is not suite approval.
- Pilot PdfCraft's bounded workflow; preserve its tests and desktop/web gates.
- A second app demonstrates the same contract before a shared runtime is mandatory.
- Distribution remains separate: [RFC-0042](0042-apple-distribution-lanes.md).

No signing, enrollment, upload, App Review submission, Android policy exception or
production migration is performed by accepting the proposal text alone.
