# RFC-0042: Opt-in Apple validation and separate distribution lanes

Status: Proposed; RFC number provisional until maintainers confirm it.

Related: [RFC-0006](0006-coding-agents.md),
[RFC-0039 / storytold/craft-contrib#60](https://github.com/storytold/craft-contrib/pull/60),
[RFC-0041](0041-native-apple-shells.md).

## Summary

Use shared, opt-in Apple build/test/package automation, then separately approved
Developer ID, TestFlight and App Store lanes. Use PdfCraft as a reference POC,
not as evidence that its current desktop bundle qualifies for the Mac App Store.
The proposed living contract is [Apple distribution](../standards/apple-distribution.md).

## Existing work and why a new decision is limited

Existing core rules already put CI in `craft-common-actions`; do not restate that
as a new rule or add app-local workflow implementations.
[storytold/craft-common-actions#259](https://github.com/storytold/craft-common-actions/issues/259)
and [its existing draft #260](https://github.com/storytold/craft-common-actions/pull/260)
own the concrete opt-in workflow and future signing proposal.

What is missing is the Apple-specific acceptance boundary: an unsigned artifact,
a Developer ID notarized DMG, a TestFlight build and a Store-reviewed app are not
interchangeable. This proposal defines that boundary without changing existing
desktop releases or authorizing publication.

## Lanes

1. **Unsigned validation:** pinned available Xcode/SDK/Rust, real desktop build,
   universal iOS app, separate iPhone/iPad tests, exact profile/target matching and
   retained test evidence. No secrets for PR/fork code.
2. **Unsigned packaging:** desktop ad-hoc-signed local artifacts, simulator app ZIP,
   and unsigned iOS-device archive. An unsigned archive is not an installable IPA.
3. **Existing macOS Developer ID:** preserve signing, hardened runtime, notarization
   and Gatekeeper checks. This remains a direct-download lane, not Store submission.
4. **Internal TestFlight:** authorized identity/profile, signed Release build,
   processing/build-ID checks and an explicitly controlled private group.
5. **External TestFlight:** invitation-only by default; Beta App Review as required.
   A public invitation link is a separate choice.
6. **Production App Review:** complete disclosures/assets, approved processed build
   and separate submission approval. Prefer manual first release after approval.
7. **Mac App Store:** a distinct sandboxed, self-contained product configuration
   with Store-managed updates and its own readiness evidence.

## PdfCraft audit and implementation boundary

The [readiness issue](https://github.com/storytold/pdfcraft/issues/7540) and
[reference app review](https://github.com/blackoutsecure/pdfcraft/blob/feat/apple-native-poc-pipeline/docs/apple-platform-review.md)
record inspected source and remaining gates.

Its current desktop entitlements intentionally omit App Sandbox; the packager
builds a DMG and separate CLI; the UI has a `craft-update` flow. Apple guideline
2.4.5 therefore prevents claiming that the current package is Store-ready.
Do not silently enable a sandbox entitlement or disable the updater in all builds:
first implement/test an explicit Store channel, scoped file/bookmark/container
behavior, authorized helpers and Store-only updates. Keep direct-download behavior.

The iOS POC's former Release-to-Debug Rust library link is an implementation defect
to fix and test, not a reason to describe a Debug archive as production Release.
Neither build correctness nor a POC disclaimer satisfies App Review completeness.

## User experience and risk

Store builds obtain durable file access through user grants/bookmarks rather than
unrestricted paths. Revoked grants must be recoverable and visible. Updates arrive
through Apple rather than an in-app installer. CLI/helper/plugin functionality may
need a separately distributed desktop product or an explicitly supported alternative;
marketing and capability discovery must disclose the difference.

Do not reduce desktop functionality globally to make the Store lane fit.
Do not promise Store parity before sandbox, input, accessibility and lifecycle tests.

## Rollout

Accept/reconcile the relevant mobile policy first; review shared CI independently;
prove unsigned builds/packages; close app-specific readiness gaps; then authorized
owners provision accounts/credentials and explicitly approve each distribution step.

RFC-0000's maintainer decision process still applies. No formal approval, purchase,
account enrollment, secret creation, binary upload or release is granted by this POC.
