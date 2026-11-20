# RFC-0039: Native Android and iOS Builds for Craft Applications

Status: Proposed

Related: [RFC-0000](0000-decision-process.md),
[RFC-0006](0006-coding-agents.md), [RFC-0013](0013-amending-rfcs.md),
[RFC-0014](0014-accessibility.md), [RFC-0015](0015-confined-file-access.md),
[RFC-0022](0022-gpu-layer.md), [RFC-0024](0024-cloud-documents.md), and
[RFC-0025](0025-suite-accounts-and-tokens.md).

## Summary

Add native Android and, later, iOS targets to the existing Rust Craft
applications. Begin with **PdfCraft on Android**, prove the approach through a
Stable release, then implement iOS and expand one application at a time with
its maintainers.

Each application keeps one engine, document model, command system, rendering
implementation, and supported integrations. Mobile Mode adapts the existing
egui interface and input handling. It is enabled by default on mobile devices,
including tablets, with a persistent toggle to the full desktop interface.
Changing modes must preserve the active document and its undo history.

Each app produces its own signed Android APK through shared CI, attached to
the same GitHub release as its desktop artifacts and built from the same source
revision. Beta can disclose mobile limitations; Stable requires demonstrated
functional parity and safety. Initial distribution is free. Store publication,
pricing, funding, and contributor compensation remain separate decisions.

This RFC requests review of the direction, feasibility, ownership, and rollout.
It does not claim that mobile builds or the referenced shared infrastructure
already work. It introduces no exception to the linked security or Rust-native
rules and does not amend accepted RFC text.

## Motivation

Phones and tablets would make PDF reading, annotation, editing, and export
available to people away from a desktop. Stylus-enabled tablets and
Chromebook-style Android devices also benefit from larger controls while
retaining keyboard, mouse, and trackpad workflows.

A separate mobile engine would duplicate format handling, commands, recovery,
and security fixes. Sharing the existing Rust implementation keeps documents
and behavior consistent. Responsive egui layouts let the project improve touch
usability without maintaining a second product interface in another framework.

The goal is the full application on mobile. A scanner or presentation remote
may be useful later, but cannot substitute for it.

## Repository evidence and existing work

The initial source review on 15 November 2026 inspected these default-branch
revisions. These are evidence snapshots, not prescribed dependency upgrades:

| Repository | Revision inspected | Findings |
|---|---|---|
| `storytold/craft-contrib` | `f7502233f8e0c8e40d75bc4afd717a4232b0bd89` | RFC process, canonical house rules, accessibility, confinement, GPU, cloud, and account proposals |
| `storytold/pdfcraft` | `17fc47b6927bf787aef2c56cf59c365e64574906` | Rust workspace, separate engine/platform/UI/automation layers, egui frontend, desktop and web entry points |
| `storytold/craft-libs` | `d181e08d3df8ff1c9b216778d668e6071a42993b` | Existing shared UI, menu, platform, print, and update crates; no `mobile`, `confine`, `cloud`, `gpu`, or `suite-account` directory in the inspected crates listing |
| `storytold/craft-common-actions` | `5b4eeb23235b9457c926c91d1228a32f7492eb2c` | Shared desktop/web build and release architecture; Android SDK cleanup in existing jobs must be considered before reusing them for mobile |

PdfCraft's [workspace manifest](https://github.com/storytold/pdfcraft/blob/17fc47b6927bf787aef2c56cf59c365e64574906/Cargo.toml),
[application manifest](https://github.com/storytold/pdfcraft/blob/17fc47b6927bf787aef2c56cf59c365e64574906/apps/pdfcraft/Cargo.toml),
[entry point](https://github.com/storytold/pdfcraft/blob/17fc47b6927bf787aef2c56cf59c365e64574906/apps/pdfcraft/src/main.rs),
[architecture](https://github.com/storytold/pdfcraft/blob/17fc47b6927bf787aef2c56cf59c365e64574906/docs/architecture.md),
and [automation documentation](https://github.com/storytold/pdfcraft/blob/17fc47b6927bf787aef2c56cf59c365e64574906/crates/automation/README.md)
provide a useful starting point. Its command registry lives in
`crates/engine/src/commands.rs`; automation tools link back to command IDs.
Counts in older architecture prose must not replace an inventory generated
from the implementation under test.

Concrete desktop assumptions requiring investigation include X11/Wayland
features in the application manifest, command-line launch and single-instance
handling, path-based document opening, XDG/home-directory settings placement,
system dialogs, printing, logging, and the opt-in loopback UI control channel.
Android content URIs cannot simply be decoded into ordinary paths. The current
rendering design rasterizes PDF pages on the CPU and presents them through the
egui graphics backend; adding a mobile target does not justify replacing it.

Existing requests and open proposals were found during this review:

- [PdfCraft #2085](https://github.com/storytold/pdfcraft/issues/2085) requests Android and iOS support, particularly tablets.
- [PdfCraft #1119](https://github.com/storytold/pdfcraft/pull/1119) proposes a Tauri/WASM mobile host. It is unmerged and differs from this RFC's Rust-native egui approach. Its document-provider and recovery work is relevant review context, not proof of this architecture.
- [craft-libs #632](https://github.com/storytold/craft-libs/pull/632) proposes `craft-mobile` with Swift/Kotlin shells and explicitly requests decisions about unsafe code and Rust-only rules. It is unmerged. This RFC instead starts with app-local adapters and no mandatory mobile crate.
- [PhotoCraft #11705](https://github.com/storytold/photocraft/pull/11705), [PhotoCraft #6931](https://github.com/storytold/photocraft/pull/6931), [LightCraft #818](https://github.com/storytold/lightcraft/pull/818), and [FilmCraft #4457](https://github.com/storytold/filmcraft/pull/4457) contain related mobile proposals. Maintainers should reconcile direction and reuse suitable work before another implementation begins.
- [craft-launcher #330](https://github.com/storytold/craft-launcher/issues/330), [PhotoCraft #1512](https://github.com/storytold/photocraft/issues/1512), and [VectorCraft #449](https://github.com/storytold/vectorcraft/issues/449) show related suite and touch-interface demand.
- [PdfCraft #6746](https://github.com/storytold/pdfcraft/issues/6746) tracks camera scanning separately; it is not a prerequisite for the full mobile port.

This is a preliminary source and proposal review, not a completed dependency
portability audit. No Android/iOS compilation, installation, hardware testing,
screen-reader testing, or runtime security validation is claimed here. Before
implementation, refresh this evidence, read current repository instructions
and issue discussions, check recent merged work, and record the exact engine,
egui/winit, SDK/NDK, bindings, renderer, and dependency revisions. Acceptance of
an RFC does not establish that its implementation has merged or is adopted by
PdfCraft. Interface names below describe dependencies to verify, not invented
APIs to call.

## Proposal

### One application with platform adapters

Preserve the current application structure and layer boundaries. Shared code
owns document parsing/editing/export, command definitions, rendering, undo/redo,
recovery logic, accessibility semantics, integrations, and validation.

App-local adapters own mobile lifecycle, orientation and safe areas, touch and
stylus translation, authorized file handles, secure storage, notifications,
system sharing, packaging, and permissions. Platform adapters may need their
own target-specific entry points or manifests; they must invoke the same engine
and UI rather than fork their functionality.

There is no mandatory shared mobile crate. Reuse existing shared libraries
where appropriate, but propose a new common abstraction only when multiple
applications demonstrate the need. This preference must be discussed alongside
craft-libs #632 rather than treated as a decision already made for that PR.

Prefer Rust and maintained platform bindings. Minimal platform glue, unsafe
exceptions, or dependency licensing exceptions require documented rationale
and the maintainer review required by the existing rules before adoption.
This RFC grants none. Do not introduce Tauri, Electron, a webview replacement,
or handwritten JavaScript/TypeScript into native apps. Preserve desktop and
existing web builds. Keep implementation and shared-CI changes independently
reviewable.

### Mobile Mode and input

Enable Mobile Mode by default on Android and iOS, on phones and tablets alike.
Persist the user's choice per app and device. A discoverable toggle restores
the existing desktop interface even on a small phone, with scrolling where
needed. Switching modes must not reload the document or reset selection,
unsaved changes, or history.

Mobile Mode provides portrait-first navigation, a bottom toolbar, contextual
panels, collapsible or scrollable tools, responsive dialogs/settings, safe-area
insets, and a searchable command palette for advanced actions. Landscape can
use a layout closer to desktop. Retain keyboard shortcuts and mouse/trackpad
support. Start with one application window; multiwindow and external-display
layouts are not pilot requirements.

Use platform touch-target guidance and accessibility scaling, documenting how
logical UI units map to device scale. Cover software-keyboard appearance,
text entry, selection, focus, orientation changes, and interrupted gestures.

Translate available hardware input into existing operations: pressure, tilt,
orientation, hover, eraser, palm rejection, multi-touch, pinch zoom, and external
devices. Expose configurable pen/finger behavior and accessible alternatives
to gestures. A device without usable pressure may use an explicitly identified
fallback or simulation; never report simulated values as hardware measurements.
Record unsupported capabilities instead of assuming a platform binding
exposes everything. Begin with PdfCraft annotation workflows, automated input
tests, and community device feedback before applying lessons to drawing apps.

### Platform compatibility and performance

| Target | Initial commitment | Evidence required |
|---|---|---|
| Android ARM64 | First build and release target; phones and tablets | Emulator smoke tests and real phone/tablet validation |
| Android ARMv7 | Conditional Beta target where dependencies permit | Dependency audit and real-device compatibility before any Stable claim |
| Chromebook-style Android | Pilot compatibility target | Runtime, input, graphics, file-provider, and ABI validation; ARM support alone proves no x86 compatibility |
| iOS/iPadOS | Implementation after Android Stable | Apple toolchain/lifecycle feasibility, signed registered-device testing, equivalent parity and security validation |
| Existing desktop/web | Preserve supported behavior | Existing repository build and quality gates |

Target devices with at least 4 GB RAM initially. Determine minimum supported
Android versions from dependencies and tests rather than promising a version
before the build proof. Support older versions where practical and disclose
the actual supported OS/ABI matrix.

Keep the existing renderer and follow RFC-0022's capability model when its
interfaces are available. Verify `craft-gpu`, wgpu, and null-backend availability
before integration; this RFC does not require waiting for a nonexistent crate
to perform a build experiment. Query the created device, choose supported
fallbacks, and recheck capabilities after device loss. Preserve document state
through graphics recovery. Use null-backend CI where applicable and real
backends on supported devices.

Bound document processing and caches through streaming, chunking, and
backpressure where applicable. Avoid making whole-document residency a new
mobile requirement; audit existing allocations and record blockers. Save
recovery state during normal work, not only on a suspension callback that the
OS may never deliver. Test memory pressure, suspension, forced termination,
interrupted long operations, and safe resume without silent data loss.

### Documents, recovery, and sharing

Use system file pickers, scoped document handles/URIs, and user-granted folders
where supported. Persist permissions only through supported OS mechanisms and
provide a way to inspect and revoke grants. Do not request broad filesystem
access to imitate desktop paths. Cached local copies do not grant authority to
overwrite the original or its neighboring documents.

Preserve editing of original documents using the provider's safest supported
save operation: retain a recoverable copy or revision, stage modified bytes
when possible, validate the result, then commit or replace through the provider.
Keep recovery data and dirty state until persistence is confirmed. Document
non-atomic alternatives and test failure at each boundary, including permission
revocation and provider disconnection. Preserve existing protections for signed
and encrypted documents; backups must not expose decrypted contents.

Version document/settings migrations, back up before migration, and preserve
recoverability across upgrades. Because Beta and Stable share app identity,
test their upgrade/migration path explicitly; an older Stable version is not
automatically a safe rollback from a newer Beta schema.

For cloud storage, follow RFC-0024's provider-neutral contracts and declared
conditional-write guarantees. A stale save must preserve local edits and the
newer remote version. Create a clearly named conflict copy where permitted,
notify the user, and retain both versions until resolution. Never label a
document remotely saved before confirmed persistence. Permit cached offline
editing where supported and synchronize on reconnect using version checks.
Do not claim atomic replacement when a provider offers only a weaker guarantee.

Preserve online functionality the desktop app actually supports. For native
mobile builds, use direct provider APIs through supported adapters, without
requiring a hosted service. Reuse `CloudBackend`, `ShareBackend`, or their
actual merged equivalents once verified. Do not create an independent
mobile-only cloud stack or imply that proposed cloud functionality already ships.

Use OS sharing for cross-app document/export handoff, with temporary,
permission-scoped access. Receiving apps must not gain unrestricted filesystem
access or credentials.

### Accounts and credentials

Follow RFC-0025's provider-neutral account and token architecture as it becomes
available. Use Android Keystore-backed credential protection and appropriate
Apple Keychain capabilities. Keep secrets out of settings, logs, crash reports,
metadata, and shared account lists. Never replace secure storage with plaintext
to unblock a port.

Share identity and sessions only through secure platform mechanisms. Use
protected per-app sessions where cross-app sharing cannot be established.
Do not emulate a desktop suite folder by reading another app's private files.
Preserve generation checks, removal-versus-refresh behavior, and the distinction
between local forgetting and provider-side revocation. Record any required
changes to the existing account contract for review under RFC-0013 rather than
silently weakening its guarantees.

### Automation and security

Mobile automation is off by default. Any eventual enablement requires explicit
per-app authorization for the current session, with expiring credentials and
only the authorized documents and operations available.

A local loopback bridge is an investigation, not an approved transport.
RFC-0015 refuses listening sockets in confined processes. Loopback binding and
authentication alone do not satisfy that rule. PdfCraft's existing desktop
control channel is not evidence that a confined mobile bridge is permitted.

Before enabling external control, document the threat model and determine
whether a separately confined external bridge can preserve the existing
contract. The receiving engine must enforce the same grants and refusals;
an unrestricted engine behind a restricted bridge is insufficient. Fail closed
when confinement cannot be verified. No public network listener is permitted.

If any design changes RFC-0015's guarantees or root/handle contract, submit a
separate explicit amendment under RFC-0013, with the required backlink, before
implementing that change. This RFC does not make that amendment. Keep automation
disabled while unresolved and record any resulting Stable parity blocker for
maintainer review; approval cannot be described as security validation.

Adapt escape tests to document handles, revoked grants, provider redirects,
linked resources, read/write separation, and OS-specific provider behavior.
Check generic/batch command routes as well as direct tools. Test refusal of
unauthorized scripts, plugins, subprocesses, network access, and unrelated
files. OS sandboxing complements engine enforcement; it does not replace it.

### Accessibility

Apply RFC-0014 and the canonical accessibility standards, including keyboard
access, accessible roles/names/states/focus, text scaling, contrast, non-color
indicators, alternatives to dragging/gestures, and announcements for saves,
errors, and completed operations. Drawing and document surfaces need usable
semantics, selection, and alternative actions.

Use existing `egui_kittest` and accessibility-tree checks where available.
Investigate actual Android and iOS semantics delivery to TalkBack and VoiceOver,
and record framework/platform gaps. A tree test alone does not demonstrate
screen-reader compatibility. Retain the required Accessibility section in UI
PRs and distinguish automated coverage from manual assistive-technology results.
Follow existing review rules rather than creating a competing mobile policy.

### Android packaging, signing, and updates

Each app has its own package identity and signed APK. Reuse established IDs;
PdfCraft already uses `ai.storyteller.pdfcraft` as its desktop identifier, which
is a candidate subject to verification of any established Android identity.
Changing an established ID requires a migration plan. Standardize new IDs with
maintainers before the first public installation.

Produce architecture-specific APKs and a general-purpose APK where feasible,
with SHA-256 checksums, exact source SHA, supported OS/ABI information, and
installation/update instructions. A universal APK covers only the ABIs it
contains; it does not imply all Android hardware is supported.

Storytold owns production signing credentials. Separate ordinary build/test
permissions from authorized release signing and publication. Untrusted PR
jobs receive no production signing secrets. Protected release jobs must verify
artifact provenance and source revision before signing; a PR-supplied artifact
is not trusted merely because another job built it.

Designated release maintainers document key custody, protected backups,
recovery, compromise response, and supported rotation/update paths. Check
certificate identity, consistent application IDs, and monotonically increasing
version codes across releases, ABIs, and channels. Application maintainers
approve functionality; release maintainers authorize signing and publication.

Beta and Stable use the same Android application ID and compatible signing
identity, so updates replace the installed app. They cannot be installed side
by side under that ID. Channel selection and version-code ordering must prevent
unexpected downgrade attempts and explain when a Beta user can return to
Stable. Prefer promoting the tested signed artifact unchanged. If source or
binary bytes change, validate the new artifact before promotion.

Build mobile artifacts from the same source revision as the corresponding
desktop version and attach them to the application's tagged GitHub release.
PdfCraft currently builds drafts from its `release` branch and creates the tag
on publication. Integrate with that release lifecycle; do not assume a tag-push
trigger already exists. Shared-CI review must settle how mobile Beta status is
represented alongside desktop assets without mislabeling either platform.

Optional update checking for GitHub-installed builds may offer a download.
Verify the trusted release source, published checksum, and APK signing
certificate against trusted app identity before offering installation. A
checksum from an arbitrary download location is insufficient. Reject untrusted
packages and require user-confirmed installation; never install silently.

Plan signing for future Google Play distribution before committing the update
identity. Validate compatibility between GitHub and Play installations, including
app-signing versus upload-key roles, before the first store publication. Do not
promise a seamless path before it has been tested.

### Shared CI and validation

Build on `storytold/craft-common-actions`, with each app providing its mobile
inputs. Changes to shared workflows require separate PRs and review under the
House Rules. This RFC adds no CI workflow or signing configuration.

The mobile workflow should prepare Rust targets and SDK dependencies, compile
and package APKs, run appropriate unit/integration and command-parity checks,
emulator launch/document smoke tests, supported accessibility checks, security
and document round-trip tests, and generate checksums/source metadata. Protected
release jobs then sign and publish only with explicit maintainer authorization.
An emulator pass is not real-device, stylus, GPU, or accessibility acceptance.

Before public Beta, exercise every existing PdfCraft command through the
appropriate engine, automation, or UI test mechanism. Generate an inventory
from the actual command registry and compare it with desktop at the same SHA.
Pair it with a maintained feature checklist, because registered commands alone
do not prove dialogs, workflows, or platform integrations work. Record mobile
reachability, test evidence, manual results, limitations, and blockers, with an
application maintainer responsible for keeping the record current.

| Area | Required pilot evidence |
|---|---|
| Launch and UI | Fresh install; portrait/landscape; mode toggle without document loss; persisted settings; dialogs, command search, contextual tools; software/hardware keyboard, mouse, touch, and stylus |
| Documents | Every command; existing PDFs; annotation/edit/save/export/reopen; content and supported metadata preservation; signed/encrypted files; corrupt input; interrupted save/recovery; cloud conflicts and offline behavior where integrated |
| Security | Unauthorized/revoked file grants; hostile documents; invalid credentials and refresh/removal races; confinement/session expiry if automation is enabled; untrusted update rejection |
| Accessibility | Keyboard and tree tests; scaling/contrast; focus and announcements; platform semantics and manual assistive-technology evidence, with gaps identified |
| Devices | ARM64 phone and tablet; Chromebook-style device with actual ABI recorded; ARMv7 during Beta; OS versions, GPU fallback/device loss, memory pressure, suspension, and termination |
| Releases | Signature/source/checksum verification; clean install and upgrade; Beta/Stable version ordering; migration/recovery; preserved desktop and web builds |

Beta may contain clearly disclosed mobile limitations, with every command
exercised and its actual result recorded rather than claimed as passing.
Stable requires the agreed functionality, accessibility, recovery, security,
and compatibility criteria. If a required capability cannot be provided
securely, document the blocker and obtain a maintainer decision. Do not silently
remove it, mark a skip as a pass, or claim full parity while it remains blocked.

## Rollout and deliverables

1. **Audit and RFC review.** Refresh repository/dependency and existing-work
   evidence; build the platform matrix; identify blockers and required
   exceptions/amendments; name pilot and release owners. Obtain maintainer
   review before adopting suite-wide policy.
2. **Android build proof.** Produce a locally installable PdfCraft ARM64 build
   from the existing engine and egui source. Verify launch, rendering, document
   access, and desktop/web preservation. Record exact build dependencies and
   instructions. This proof is not a public Beta.
3. **Mobile Mode.** Add responsive layouts, mode persistence, input handling,
   and access to every command through appropriate controls or the palette.
4. **Files, cloud, and security.** Integrate scoped access and recovery, then
   actual supported provider/account interfaces. Document the threat model,
   test conflicts and revoked access, and resolve confinement before external
   control is enabled.
5. **Validation.** Complete generated command inventory and feature evidence,
   automated tests, real-device testing, accessibility review, document
   integrity, and Beta blocker resolution.
6. **Release pipeline.** Independently review shared CI, protected signing,
   and Beta/Stable procedures. Publish authorized Beta APKs alongside the
   application's release; test updates and promote only with maintainer
   approval and the Stable criteria satisfied.
7. **iOS.** After Android Stable, reuse the engine and Mobile Mode. Implement
   lifecycle, File Provider/document access, Pencil input where supported,
   Keychain, sharing, accessibility, graphics, and memory adapters. Establish
   signing/provisioning and equivalent validation. Produce signed IPA builds
   for registered testing devices using supported Apple distribution processes,
   with GitHub artifacts/instructions where appropriate. Do not claim arbitrary
   iPhones can install an IPA directly from GitHub. App Store publication is a
   separate approval.
8. **Suite expansion.** Publish the lessons and migration checklist. Select
   the next app with its maintainers and repeat without forking the engine.
   Existing mobile contributors should be included in that coordination.

The audit, architecture/platform matrix, build instructions, UI/input changes,
file/cloud/account integration, threat model/confinement evidence, parity
inventory, signing/CI documentation, release procedures, iOS design, and
migration guide land in the repositories that own them through small PRs.
They are engineering deliverables for those phases, not completed artifacts
or prerequisites to discussing this RFC.

Report affected repositories, changes, exact builds/tests run, devices and OS
versions, unverified behavior, blockers, deviations, and required maintainer
decisions at each stage. Do not claim implementation or hardware success from
source inspection alone.

The PdfCraft pilot succeeds when the Android app builds from the existing Rust
source, desktop/web still work, both UI modes preserve the document, commands
remain usable, documents round-trip and recover safely, cloud conflicts retain
both versions where supported, permissions/confinement meet approved boundaries,
accessibility and required tests meet the existing standards, signed installs
and upgrades work, and a maintainer approves Stable.

Verify the current application inventory before expansion. Candidates include
PhotoCraft, VectorCraft, LightCraft, FilmCraft, EffectCraft, SoundCraft,
WordCraft, GridCraft, DeckCraft, DesignCraft, FigCraft, and CADCraft; this is not
an exhaustive inventory or a committed migration order. Each needs its own
adapters, package identity/APK, command and feature evidence, accessibility,
app-relevant hardware tests, security/document round trips, and approval.

## Distribution and funding

Initial GitHub APK distribution is free, and initial Android/iOS store
distribution is intended to be free. Store publisher accounts should be
organization-owned. This work introduces no paid feature restrictions.

Donations, contributor funding, and any later nominal store price (for example
$0.99) require separate discussion and governance decisions. This RFC allocates
no revenue, establishes no compensation model, and authorizes no account fees
or store publication.

## Alternatives

- **Separate mobile engines or codebases.** They allow independent development
  but duplicate format, recovery, command, and security maintenance.
- **Tauri/Electron/webview hosts.** Existing proposals provide relevant
  experiments, but this RFC chooses the native Rust/egui path within the
  current app architecture and rules.
- **Native SwiftUI/Compose or React Native frontends.** They may offer platform
  UI benefits but introduce another interface implementation and require
  policy and maintenance decisions beyond this proposal.
- **Mandatory shared mobile kit first.** Common lifecycle/input work may
  eventually justify extraction, but requiring a new crate before app-local
  proofs risks fixing the abstraction too early. Review existing craft-mobile
  work before deciding that boundary.
- **Web/PWA only.** Preserve and improve the web build, but it does not by
  itself establish native packaging, scoped provider access, or native input
  behavior.
- **Simplified companion apps.** Useful as separate proposals, but they do
  not meet the requested full-application parity goal.
- **Port the whole suite together.** This amplifies unverified dependency and
  UI assumptions. A PdfCraft pilot lets maintainers evaluate evidence first.

## Open questions and maintainer decisions

- Is this native Rust/egui, PdfCraft-first direction worth adopting suite-wide,
  and how should it be reconciled with the existing mobile PRs and owners?
- Who maintains the pilot, parity evidence, device matrix, and shared release
  workflow? Which dependencies currently block Android and later iOS?
- Are app-local adapters sufficient with maintained safe bindings? What minimal
  platform glue, if any, needs a documented exception and independent review?
- What minimum Android version, ABIs, device coverage, and performance budgets
  can the build proof and available testers support?
- Which secure credential-sharing mechanism can satisfy the account contract
  on each OS, and which apps must use protected per-app sessions?
- Can mobile document grants and an external automation bridge satisfy
  RFC-0015 unchanged? If not, what separate amendment should be reviewed?
- Which checks block Beta and Stable, and how should mobile channel status
  appear on a release shared with desktop artifacts?
- Who owns signing and recovery, how are version codes allocated, and what
  signing arrangement preserves a tested future GitHub/Play update path?
- When is iOS feasibility sufficient to fund organization-owned signing/store
  accounts? Store publishing and any pricing remain separate approvals.

Review follows RFC-0000: a maintainer starts the comment period and another
maintainer's approval is required with no unresolved maintainer objections,
subject to the process's owner decisions. This proposal adds platform targets
to existing apps and requests the ordinary three-day period; it does not set
the start or end date itself. Any required amendment follows RFC-0013.
