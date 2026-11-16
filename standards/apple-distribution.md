# Apple validation and distribution

Proposed by [RFC-0042](../rfcs/0042-apple-distribution-lanes.md); not yet suite policy.

## Reusable local/CI interface

The reference app exposes:

```sh
bash scripts/build-apple.sh build all Debug
bash scripts/build-apple.sh test all Debug
bash scripts/build-apple.sh package all Release
```

The positional contract is `<build|test|package>
<all|macos|ios-simulator|ios-device> <Debug|Release>`.
App-owned commands contain app-specific engine, native project and package names.
Shared CI owns toolchain/runtime selection, device setup, evidence and artifact
handling. No `eval`, caller-supplied shell command or code scraping is necessary.

The reference test driver accepts distinct `CRAFT_IPHONE_SIMULATOR_UDID` and
`CRAFT_IPAD_SIMULATOR_UDID`; shared setup selects actual device types from an
available iOS runtime. Run families sequentially when bindings/DerivedData are
shared locally. CI jobs must isolate worktrees/output or use the same sequencing.
The current POC supports Debug native tests; unsupported combinations fail clearly.
Unsigned device compilation must never report a physical-device test.

Build outputs retain each app's existing layout. Package outputs are below
`target/apple/<Debug|Release>/`. Verify actual products, platforms, architecture,
device families and matching Rust/native profiles; upload failures or absent
required products fail the job. Synthetic-consumer tests establish orchestration
reuse, not hosted Apple compilation or application behavior.

## Source and build integrity

- Record the exact source SHA and dirty-state boundary, dependency lockfiles,
  binding-generator version, Rust target/profile, Xcode/SDK and tested runtime.
- Use an available supported runner/Xcode pair. Do not assume the newest local
  Xcode exists on GitHub-hosted images; fail on a missing runtime instead of quietly
  substituting a different test matrix. Recheck Apple's upload requirements.
- A device Rust archive targets iOS, not the identically named CPU's Simulator ABI.
  Native Release must link Rust Release and generate bindings from that build.
- Reuse canonical artwork, attribution/notices and compatible desktop metadata.
  Allocate a separate monotonically increasing Store build number; the marketing
  version is not a sufficient CI upload counter.
- Keep unsigned validation opt-in initially. Do not introduce new default macOS
  runner cost or change current desktop release schedules as a feature-PR side effect.

## Credentials and approvals

An authorized organization's Account Holder owns enrollment, legal agreements,
seller identity, bundle IDs and App Store Connect records. Simulator work requires
no paid membership. Do not use a fork/contributor's seller account implicitly.

Create signing certificates/profiles in Apple Developer Certificates, Identifiers
& Profiles and upload/API access in App Store Connect Users and Access. A `.p12`
needs its private key; a `.cer` alone is not enough. An upload `.p8` is a different
private key, not a signing certificate.

Keep private materials in the consuming app repository's protected GitHub
Environment/organization secret store, never source, issues, artifacts or the
shared-actions repository. Use environment-specific, explicitly mapped secrets,
approved refs/reviewers, least-privilege access, temporary keychains and guaranteed
cleanup. Base64 is transport encoding, not encryption. Team API-key permissions
can span apps despite GitHub repository scoping.

Developer ID, iOS distribution and Mac App Store application/installer identities
and provisioning requirements differ. Review actual certificate type, profile,
entitlements, bundle/team identifiers and export method for the selected lane.
Do not infer Store eligibility from successful notarization.

## Staged testing and release

For either iOS or supported macOS TestFlight, record the processed Apple build ID
before assigning groups. Disable automatic pilot-group distribution and audit
existing automatic assignments before upload. Internal groups are authorized App
Store Connect users; invitation-only external testing is different from internal
testing and may require Beta App Review. Public TestFlight links are not public
App Store releases. Builds expire after 90 days under current Apple rules.

Do not choose an internal-only build restriction if the same build must reach
external testers or customers. Separate upload, invitations/group assignment,
external beta review, production submission and release approvals. Prefer manual
initial public release; beta approval is not production approval.

## Store readiness gates

Record pass/blocker/not-verified with source or runtime evidence, not an invented
compliance score. Apple alone decides acceptance.

- Functional completeness, accurate supported-feature disclosures and screenshots.
- Real hardware/current and minimum supported OS, lifecycle/memory, accessibility
  and provider tests; no reliance on Simulator labels as full VoiceOver evidence.
- Privacy/required-reason API manifests including relevant transitive SDKs, public
  privacy/support URLs, age rating, legal/export-compliance and licensing review.
  PDF encryption means an owner must assess export compliance, not guess an exemption.
- For Mac App Store: App Sandbox, scoped file/bookmark/container migration,
  self-contained Xcode submission, no unauthorized external installers/resources,
  no forbidden privilege or persistent helper behavior, Store-only updates and
  all claimed localizations in the bundle (guideline 2.4.5).
- For native mobile: scene/input/background behavior, Files/provider integration,
  no external executable plug-ins/self-updater, and complete authorized signing.

## Official references

Reviewed 2026-11-16; check again before distribution:

- [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/):
  completeness/metadata 2.1/2.3, Mac distribution 2.4.5, software/Files 2.5,
  design/minimum functionality 4 and privacy 5.1.
- [App Sandbox](https://developer.apple.com/documentation/security/app-sandbox)
  and [file access](https://developer.apple.com/documentation/security/accessing-files-from-the-macos-app-sandbox).
- [Upload builds](https://developer.apple.com/help/app-store-connect/manage-builds/upload-builds/)
  and [current requirements](https://developer.apple.com/news/upcoming-requirements/).
- [TestFlight](https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview/).
- [Required-reason APIs](https://developer.apple.com/documentation/bundleresources/describing-use-of-required-reason-api)
  and [third-party SDK requirements](https://developer.apple.com/support/third-party-SDK-requirements/).
