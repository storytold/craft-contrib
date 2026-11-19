# RFC-0041: Microsoft Purview/MIP prerequisites, rollout, and renewal standard

## Status

Draft. This document defines the suite-wide operating standard for Microsoft
Purview/MIP integration. It does not by itself claim that a public release,
tenant test, or application integration is complete.

Discussion: [storytold/craft-contrib#73](https://github.com/storytold/craft-contrib/issues/73)

## Summary

One Microsoft Entra app identity and one public-application IPIA may serve the
native Craft application suite. The MIP SDK and redistributables are bundled
per application and platform, while each customer tenant owns its own
Purview licensing, sensitivity labels, policies, consent, and recipient
authorization.

This RFC does two things:

1. States the exact Storytold prerequisites for public Microsoft Purview/MIP
   support.
2. Defines what must be done once, what is repeated per app or platform, what
   triggers renewal or retesting, and what happens if a prerequisite is not
   completed.

PdfCraft is the implementation POC and the completion standard for PDF:

- Feature issue:
  [storytold/pdfcraft#6747](https://github.com/storytold/pdfcraft/issues/6747)
- Implementation PR:
  [storytold/pdfcraft#7843](https://github.com/storytold/pdfcraft/pull/7843)
- POC follow-up commit:
  [dbde699256f5594cdf755651de5363c40eba0c4a](https://github.com/storytold/pdfcraft/commit/dbde699256f5594cdf755651de5363c40eba0c4a)

The POC currently proves native SDK bridging, PFILE detection, rights and
save-state seams, fail-closed behavior, password/MIP composition, UI,
automation/MCP, and packaging. It does not claim live tenant authorization
until the complete external test gate in this RFC passes.

## Motivation

Purview/MIP support crosses several boundaries that cannot be solved by one
application PR:

- Microsoft Entra app identity and delegated API consent.
- Microsoft legal/commercial integration approval.
- Official MIP SDK binaries and redistributable licenses.
- Platform signing and notarization.
- Customer tenant licensing, label policies, MFA/Conditional Access, and
  recipient rights.
- Format-specific package handling and rights enforcement in each app.
- Repeatable validation after SDK, OS, app, or policy changes.

The suite needs one stable trust boundary, without pretending that a file
format pass-through or a local SDK build proves public or tenant support.

## Scope

This RFC applies to:

- PdfCraft.
- GridCraft.
- DeckCraft.
- WordCraft.
- ArtCraft and generated-PDF/Office export flows.
- Any future native desktop Craft app that reads, labels, protects, exports,
  or republishes Microsoft-supported documents.
- Shared providers or package work that lives in `storytold/craft-libs`.
- Release tooling that packages MIP SDK helpers or redistributables.

This RFC does not make every app MIP-capable. Each application needs an
approved format adapter and rights implementation. Package metadata
preservation is not treated as live MIP support.

## Definitions

- **Suite app identity**: The one public/native Microsoft Entra application
  registration used by the Storytold Craft desktop suite.
- **IPIA**: Microsoft Information Protection Integration Agreement. Microsoft
  requires this formal agreement before a public application developed with
  MIP SDK is released.
- **MIP SDK package**: The official Microsoft C++ SDK package and headers,
  binaries, license/redistribution files, and supported platform
  redistributables obtained from Microsoft's official distribution channels.
- **Helper**: The signed native process or library bridge that calls the MIP
  SDK on behalf of one Craft app.
- **MIP native protection**: Protection that is enforced by the associated
  application and SDK for supported text, image, Office, and PDF file types.
- **Generic protection**: `.pfile` encapsulation for file types not supported
  natively. Microsoft documents that generic protection authenticates access
  but does not enforce usage rights inside the app; rights are informational
  rather than a native usage-rights enforcement mechanism.
- **PFILE**: A generic `.pfile` protected container or PFILE-style protected
  PDF payload. It must not be opened as plaintext.
- **Customer tenant**: A Microsoft 365/Purview tenant owned and administered by
  a customer. Storytold must not provision or silently administer the customer's
  labels or Purview policy.
- **One-time prerequisite**: Required once for the suite trust boundary unless
  the trust boundary or legal/publisher identity changes.
- **Repeated prerequisite**: Required again because of SDK, app, platform,
  certificate, customer, consent, policy, or security-change triggers.

## Standard decision

Storytold will maintain:

1. One multitenant public/native Entra app registration for the Storytold
   Craft suite.
2. One IPIA associated with the suite app identity.
3. One official-SDK build and validation procedure per supported app/platform.
4. Separate app/platform signing and package procedures.
5. A format-specific rights and re-publishing implementation in each app.
6. One suite-wide test matrix and evidence standard.

Separate app IDs are allowed only for a real trust-boundary change, such as a
separate publisher/legal owner, separate public consent/revocation boundary,
or a third-party/customer fork released under a different trust name.
Different operating systems, apps in the suite, customer tenants, or test
environments are not reasons to create a new public app ID.

If a separate app ID is created, Microsoft requires the IPIA/update process to
cover that app identity. An existing IPIA must not be assumed to cover it.

## Suite-wide prerequisites

| Requirement | Owner | Frequency | Exact procedure | If completed | If skipped |
|---|---|---|---|---|---|
| Multitenant public/native Entra app registration | Storytold security + Entra administrator | Once for the suite app identity; update only on permission/redirect/trust-boundary change | Register an app named `Storytold Craft Applications` or another stable approved public name. Set supported account types to multitenant organizational directories. Configure `http://localhost` as a mobile/desktop redirect and allow public client flows. Do not add app secrets or application certificates to the desktop identity. | Apps can request delegated tokens and customer tenants can grant consent. | User sign-in cannot proceed; protected PFILE/IRM opens fail closed with not-configured or sign-in-required state. |
| Delegated MIP API permissions | Storytold Entra/security owner | Once; re-review on permission or consent-policy change | Add delegated `Azure Rights Management Services/user_impersonation` and `Microsoft Information Protection Sync Service/UnifiedPolicy.User.Read`. Do not grant Content.SuperUser, Content.DelegatedReader/Writer, UnifiedPolicy.Tenant.Read, or application-only permissions for normal desktop use unless a separate service/daemon app has a documented need. | SDK policy and rights calls receive delegated authorization. | Label discovery, use-license acquisition, or label operations fail or return incomplete rights. |
| Public IPIA | Storytold legal/business owner | Initial for the public suite app identity; update for a new app ID/publisher/trust boundary or a Microsoft-directed change | Email `IPIA@microsoft.com` with the application/product name, requester identity, and contact email. Microsoft returns a form. Return the legal company name, incorporation location/country, URL, contact, app name, description, Azure tenant ID, App ID, and critical-support contacts. Digitally sign the final IPIA and wait for Microsoft signature. For a new App ID on an existing IPIA, email Microsoft with the app name, description, tenant ID, App ID, and critical-support contacts. | Storytold may distribute a public SDK-enabled app under the signed identity. | Internal POC builds may continue, but public SDK-enabled builds must not be claimed or released. |
| Official MIP SDK package | Release engineering | Every SDK adoption/release; repeat on SDK, security, OS, or architecture change | Download the official package from Microsoft's official MIP SDK channel. Verify the package, version, license, `redist.txt`, `include/`, and `bins/release/<arch>/` for each OS/architecture. Record version, SHA-256, source URL, and redist inventory. | Builds are reproducible and redistributable policy is reviewable. | SDK build fails closed or cannot be packaged. |
| Native helper and app packaging | Release engineering | Once per app/OS adapter; repeated for every SDK/app release | Build native C++ SDK helper, stage official SDK binaries, and package the helper with one app/platform release. Do not copy or hand-roll SDK binaries. | The helper starts and loads SDK dependencies. | App reports SDK unavailable or the OS blocks/quarantines the helper. |
| Platform signing/notarization | Storytold release/security owner | Every release; repeat for certificate/identity renewals | macOS: codesign nested SDK/helper/dylibs and notarize/sign the app artifact. Windows: Authenticode sign the app/helper/DLL package. Linux: sign/package according to the distribution and supported package format. Include or stage platform runtime dependencies required by the SDK. | Public OS trust checks pass. | macOS quarantine, Windows SmartScreen/Authenticode, or Linux package integrity checks block the helper. |
| Microsoft account session and account switching | Storytold identity/engineering owner | POC in-process account list; public persistence repeated for platform credential-store changes | Use Microsoft sign-in for account discovery with no email textbox. Keep the active account and its RMS/MIP tokens scoped to the in-process account session. Public builds persist account profiles and refresh/session state only in the platform credential store. | A person can add, switch, and sign out accounts, and automation can report the active account. | POC account switching may work only for the running process; public multi-account support cannot be claimed across restarts. |
| Microsoft 365 test tenant | Storytold security/engineering | Once for the POC tenant; repeat tenant-policy validation after major label/policy changes | Create a dedicated tenant or isolated test environment. Publish test labels, protected and unprotected labels, mandatory/default/downgrade policies, test users/groups, MFA/Conditional Access, and one PFILE fixture. | POC can test real use-license and rights behavior. | No live result can be claimed. Synthetic tests prove code paths only. |
| Customer tenant licensing and label policy | Customer Purview/Microsoft 365 administrator | At onboarding and whenever the customer changes Purview license, labels, policy, user scope, recipient scope, or consent policy | Customer enables the required Purview/IRM licensing and label policy, publishes labels, configures protected-label permissions and external recipients, and grants tenant consent when the tenant policy requires it. | Customer users receive tenant-approved labels and rights. | Labels are missing, sign-in works but authorization fails, or recipients cannot open protected documents. |
| App-specific format and rights implementation | Each app repository owner | Once per app format track; repeated for format/SDK/rights behavior changes | Implement format parse/write, label metadata, rights gates, UI/automation, save/re-publish behavior, private staging, and tests. Link a format-specific issue/PR to this RFC. | The app supports the format's agreed MIP behavior and cannot create an unauthorized plaintext copy. | Package metadata may survive, but live MIP sign-in, decryption, rights, or re-publishing is missing or falsely claimed. |

## Exact suite app setup procedure

### A. Storytold app registration

1. The Storytold Entra administrator opens App registrations and creates or
   confirms `Storytold Craft Applications`.
2. Set supported account types to multitenant organizational directories.
3. Add `http://localhost` as the mobile/desktop/public-client redirect.
4. Enable public client/desktop flows.
5. Add delegated `Azure Rights Management Services/user_impersonation`.
6. Add delegated
   `Microsoft Information Protection Sync Service/UnifiedPolicy.User.Read`.
7. Grant tenant-level admin consent in the Storytold tenant if Storytold
   tenant policy requires it.
8. Store the public client ID in the release manifest/CI configuration.
9. Never store a client secret, tenant secret, or private key in source,
   desktop binaries, environment files, logs, or screenshots.

The app ID is a public identifier. It is not a secret. Customers do not
register a separate Storytold app for each tenant.

### B. IPIA

1. Legal submits the IPIA request to `IPIA@microsoft.com`.
2. The request identifies the company, app name, requester, email, and
   critical-support contacts.
3. Microsoft sends the IPIA form. Storytold provides legal name, incorporation
   location/country, company URL, app name/description, Azure tenant ID,
   App ID, and contact details.
4. Both parties sign the IPIA before public SDK-enabled distribution.
5. Storytold records the app ID, IPIA version/approval date, renewal or
   change owner, and evidence location.
6. A new App ID or real publisher/trust-boundary change follows Microsoft's
   IPIA update process; it is not assumed to be covered automatically.

### C. Official SDK package

1. Release engineering selects an exact SDK version and target package.
2. Storytold obtains the package from Microsoft's official MIP SDK download
   channel.
3. It verifies source, version, license, `redist.txt`, and third-party notices.
4. It verifies:
   - macOS arm64 and x86_64 SDK/layout where supported.
   - Windows amd64, x86, and arm64 SDK/layout where supported.
   - supported Linux distribution/architecture layouts.
5. It records package SHA-256 values and redist inventory outside app source.
6. It builds the native helper in each supported app and OS target.
7. It packages only official SDK binaries and reviewed dependencies.

Official Microsoft guidance:

- [MIP SDK setup and configuration](https://learn.microsoft.com/en-us/information-protection/develop/setup-configure-mip)
- [MIP SDK API permissions](https://learn.microsoft.com/en-us/information-protection/develop/concept-api-permissions)
- [MIP SDK supported file types](https://learn.microsoft.com/en-us/information-protection/develop/concept-supported-filetypes)
- [MIP SDK overview](https://learn.microsoft.com/en-us/information-protection/develop/overview)
- [Official MIP SDK binaries](https://aka.ms/mipsdkbinaries)
- [SharePoint-compatible protected-PDF readers](https://learn.microsoft.com/en-us/purview/sp-compatible-pdf-readers-for-irm)

### D. Application build and package

1. The app enables MIP SDK support only in an explicit SDK build feature.
2. It resolves the SDK root and app ID at build/release time.
3. It stages the native helper beside or inside the app according to platform
   rules.
4. It stages official SDK libraries and required runtime dependencies.
5. It signs nested code before the parent app/package signature.
6. It notarizes macOS releases when the real signing identity is available.
7. It Authenticode-signs Windows app/helper/DLL/MSI components where required.
8. It signs or packages Linux artifacts according to the distribution target.
9. It proves helper discovery and runtime library resolution after staging,
   before user testing.

Ordinary builds remain ordinary password-protected/PDF or format builds. They
must not present MIP support as enabled and must not open MIP-shaped bytes as
plaintext.

### E. Tenant POC

1. The security/identity owner confirms the test tenant and test users.
2. The Purview administrator publishes test labels and label policies.
3. The tenant creates protected and unprotected PFILE/PDF fixtures.
4. The app opens the fixture and requests Entra sign-in if needed.
5. The SDK acquires delegated tokens, use-license, label metadata, and rights.
6. The app enforces rights and re-publish behavior.
7. The test matrix includes MFA, Conditional Access, external recipient,
   no-access, view-only, copy/print/edit/export/owner/change-rights, label
   justification, remove-label, remove-protection, save, revert, and reopen.
8. The result is recorded with pass/fail evidence and no secret or personal
   tenant data.

## Recurrence and renewal standard

These tasks repeat based on triggers. The absence of a fixed annual date does
not mean no owner or procedure.

| Task | Normal trigger | Suggested timing | Procedure | Evidence |
|---|---|---|---|---|
| SDK package/security update | New SDK release, security advisory, or app major release | Immediately for a security advisory; otherwise in the next major SDK/OS support window | Download official package, verify license/redist, rebuild all affected app/platform targets, rerun unit/tenant matrix, sign/package, update RFC/PR status | SDK version, SHA-256, test report, package report |
| App/platform signing certificate | Certificate renewal, identity change, or signing failure | Renew 60-90 days before expiry; never wait for expiration | Provision/rotate certificate, update CI, codesign helper and libraries, notarize/sign/package, verify OS trust | Certificate ID or fingerprint metadata, verification log |
| IPIA/app identity review | New App ID, separate trust boundary/publisher, legal owner change, or Microsoft request | At change time and after the relevant app identity approval | Send update/request as required, record Microsoft acknowledgment and signed IPIA/change, update app registration and package identity | IPIA record, app ID, Microsoft acknowledgment |
| Public app permission/redirect review | Permission or auth protocol change | At design review and before public release | Confirm delegated permissions only, public/native client redirect, consent behavior, no secrets, no application-only privilege | Entra change record, app manifest, permission diff |
| Customer tenant consent/licensing | Customer onboarding or tenant policy/permission change | Once per tenant when required; recheck after app permission or policy change | Verify Purview license, tenant consent, label publication, user/group scope, external recipient policy, MFA/Conditional Access | Customer check record and test fixture |
| Tenant POC matrix | SDK update, major app release, OS support change, or label-policy change | Before each public SDK-enabled release and after each relevant tenant-policy change | Run real PFILE fixtures and rights matrix; run ordinary-password and MIP composition; capture evidence | POC report, screenshots, pass/fail count |
| Microsoft account-session persistence | Account UX, auth protocol, credential-store, or app major-release change | POC before release; public builds before public multi-account support is claimed | Test add/switch/sign-out behavior, active-account isolation, OS credential-store persistence, token cleanup, app restart, and no plaintext settings/log storage | Account matrix, credential-store evidence, sign-out/restart verification |
| App format rights implementation | Format adapter or rights behavior change | Once per app rollout; rerun for format/save/export/rights changes | Add issue/PR, format tests, engine/UI/CLI/MCP gates, re-publish/revert tests, tenant POC | App PR and test evidence |
| Security and dependency review | SDK update, package change, vulnerability, or annual review | Triggered immediately for a finding; at least annually for the suite | Review SDK/C++ helper dependencies, staging cleanup, signing, logging, temp-file permissions, and revocation behavior | Security-review result and remediation link |

This is a trigger-based standard. It intentionally does not say "renew every
year" unless a legal, certificate, or Microsoft term is actually time-bound.
The owner and evidence must be recorded even when no renewal is needed.

## Outcomes and failure matrix

| Situation | App behavior | Product/release outcome |
|---|---|---|
| MIP SDK package/helper absent | Ordinary build recognizes MIP-shaped files and fails closed; SDK unavailable state is explicit | Local ordinary build and POC UI tests may run; no live Purview support is claimed |
| App ID absent | SDK helper exists but app registration is not configured | Sign-in cannot proceed; protected file reports not configured |
| App ID present but public IPIA absent | SDK can build internally for POC testing | Public SDK-enabled distribution is prohibited/unsupported until IPIA approval |
| SDK/helper/signing present but tenant consent absent | Sign-in or label/rights API returns tenant/consent error | User can retry; no protected content is opened without authorization |
| Customer Purview license/label policy absent | Labels/policy or use-license unavailable | Protected PFILE may fail authorization or labels may not appear |
| PFILE without SDK or rights state | No plaintext fallback | User receives protected-document/not-configured/SDK-unavailable state |
| PFILE with sign-in but no use-license/rights | No-access or rights-restricted state | File remains closed or view-only according to SDK-confirmed rights |
| Protected file with export/owner rights | Re-publish/remove operations proceed only through SDK authorization | App may produce SDK-authorized output; unprotected output is never a silent fallback |
| Protected file without export/owner rights | Remove protection/export is denied | App reports the rights requirement and preserves protection |
| SDK helper missing runtime dependency | OS/dynamic-loader or helper startup error | Package is not accepted for release |
| Certificate expired or app not notarized/signed | OS trust or package validation blocks helper/app | Release blocked until signing/trust is repaired |
| Labeling required and no label chosen | Save/re-publish is blocked | User gets a mandatory-label/default-label prompt or clear save failure |
| Password protection and MIP protection both apply | Effective rights are the intersection of ordinary PDF permissions and MIP rights | No operation may bypass either security layer |
| Everything above is complete | SDK signs in, acquires rights, opens authorized content, enforces operations, and re-publishes through the SDK | Public POC or app feature can be released with live tenant evidence |

Ordinary builds must remain usable and must never silently export or strip
protected content in a fallback path.

## Current application applicability

| Repository/app | Current evidence | RFC status | Remaining work |
|---|---|---|---|
| PdfCraft | [issue #6747](https://github.com/storytold/pdfcraft/issues/6747), [draft PR #7843](https://github.com/storytold/pdfcraft/pull/7843), PFILE detection, native helper, SDK build, rights/save/label/password/UI/automation/packaging seams | POC implementation is substantially complete; live tenant gate remains | Public app ID/IPIA, tenant POC, real use-license/rights, signed/notarized SDK package |
| GridCraft | [issue #865](https://github.com/storytold/gridcraft/issues/865), [open PR #1221](https://github.com/storytold/gridcraft/pull/1221) for sensitivity-label and table/custom-property pass-through | Metadata pass-through exists as format-fidelity work | Live MIP labels/protection, spreadsheet rights, re-publish/save, and tenant POC |
| DeckCraft | [merged PR #535](https://github.com/storytold/deckcraft/pull/535) for sensitivity-label/customXML pass-through; [merged PR #2176](https://github.com/storytold/deckcraft/pull/2176) for password-protected PPTX CFB detection | Metadata and ordinary encrypted-deck detection improve format compatibility | Live MIP label/protection, presentation rights, re-publish/save, and tenant POC |
| WordCraft | [issue #5827](https://github.com/storytold/wordcraft/issues/5827) for password-protected DOCX/ODT | Ordinary password/encryption work is separate | Live MIP labeling/protection for Office-supported files, rights, save/re-publish, tenant POC |
| ArtCraft | [issue #2447](https://github.com/storytold/artcraft/issues/2447) | Enterprise export/label requirement is documented | Decide supported export formats, label application, live tenant policy, rights, and automation |
| Future CAD/partner app | Microsoft supports native/protected formats such as `.dwfx` and has partner patterns such as [HaloCAD](https://www.microsoft.com/en-us/security/blog/2021/07/22/how-to-protect-your-cad-data-files-with-mip-and-halocad/) | No current app repository is accepted as live MIP work | Confirm specific file formats and SDK support, partner/plugin trust boundaries, plugin IPIA, and whether the host app needs its own IPIA |
| Future FigCraft/VectorCraft/DesignCraft/media exporters | No live implementation record | Candidate only | Adopt only when the exported format is supported and the app can preserve label/protection/rights evidence |

Microsoft's supported-file-type matrix also governs scope:

- PDF and Microsoft Office files support native protection.
- PowerPoint/Word/Excel native protection keeps or changes the relevant file
  extension according to Microsoft format rules.
- Generic protection creates a `.pfile`; it is encapsulation and
  authentication, not native usage-rights enforcement.
- Container files such as ZIP/MSG/EML/PDF attachments are not recursively
  labeled/protected; app-level recursion is the app's responsibility.

No app should describe pass-through or generic PFILE handling as complete live
native MIP support.

## App rollout procedure

Each app follows the same implementation gate:

1. Open or update an app issue that names the format, supported/unsupported
   operations, rights, and customer workflow.
2. Link the app issue/PR to this RFC and the PdfCraft POC evidence.
3. Move format-independent provider/auth/rights policy into
   `storytold/craft-libs` if it is shared.
4. Keep format parsing, editing, rights gates, and re-publishing local to the
   app.
5. Wire the engine/API, desktop UI, CLI/MCP, save/export/print/edit operations,
   and mandatory-label behavior.
6. Add ordinary-format and synthetic MIP tests.
7. Run the real tenant PFILE matrix.
8. Add platform packaging/signing evidence.
9. Attach screenshots and runtime evidence to the app PR and RFC.
10. Keep the app work draft or unsupported until the live tenant and public
    release gates pass.

## UI and operation standard

The suite will use native Rust UI components and localization, original Craft
design, existing icon conventions, and shared states. Public screens and
behavior may use Microsoft or PDF/Office behavior as a usability reference,
but not as copied artwork, trademarks, icons, proprietary fonts, screenshots,
or design traces.

### Microsoft account UX

The suite will not ask the person to type or repeat an email address before
Microsoft sign-in. The Microsoft sign-in screen owns account discovery,
selection, MFA, Conditional Access, tenant consent, and account verification.
PdfCraft passes no pretyped email field to Microsoft unless an automation/API
caller explicitly supplies a login hint.

When signed out:

- Show one primary **Sign in** action.
- Do not show **Sign out**.
- Do not show an account email textbox.
- If app configuration or the SDK is unavailable, show that state instead of
  a false sign-in action.

When signed in:

- Show the active account in the account card.
- Prefer Microsoft `display_name`; show email/principal name as the secondary
  line. A compact `Display name - email/principal name` label is the fallback
  wherever one identity label is needed.
- Replace **Sign in** with **Sign out** for the active account.
- Offer **Add account** to start Microsoft account selection without exposing
  an email entry field.
- Offer an account switcher only when more than one account is signed in.
- The active account is the only account used for the MIP helper until the
  person switches or signs out.

Multiple accounts are part of the suite account contract:

- Signing in adds or replaces the account's in-process session and makes it
  active.
- Switching account changes the active RMS and MIP policy/sync session
  immediately; open protected documents are retried only through the normal
  rights-aware reopen/re-publish path.
- Signing out removes the active account from the in-process account list.
  If another account remains, that account becomes the next active account;
  if none remain, the app becomes signed out.
- Account profile metadata and active account selection must be verifiable in
  the UI, CLI/MCP status, and automation result.
- Public builds should persist account profile and refresh/session state in
  the platform credential store (for example, macOS Keychain, Windows
  Credential Manager, or Linux Secret Service). Tokens must never be stored
  in settings files or logs.
- The POC may keep account sessions in process, but public persistence is
  required before public multi-account support is claimed.

### Detected document state

The sensitivity-label dialog is not a static label catalog. Its document
section is populated only after successful Microsoft sign-in and SDK/tenant
inspection of the open document.

For a detected label or protected file, the dialog shall show:

- The document label name and policy description or tooltip.
- Sensitivity/label metadata available from the SDK.
- Protection state: label-only, protected/IRM, or unprotected.
- Document owner and, when available, the protected-policy/content ID.
- Mandatory-label, default-label, and downgrade-justification requirements.
- A dynamic capability matrix: View, Edit, Save, Print, Copy/Extract,
  Comment, Export, and Change Security.
- Each capability's allowed/not allowed state, derived from SDK-confirmed
  rights and intersected with ordinary PDF password permissions.
- Available label operations only when the current account and policy permit
  them.

The Apply, Remove Label, and Remove Protection actions are stateful:

- Apply is available only when the account is signed in and a tenant label is
  selected.
- Remove Label is available only when a current label exists and the SDK/rights
  state permits changing security.
- Remove Protection is available only when the document is protected and the
  SDK confirms EXPORT/Owner or Change Rights capability.
- Justification is required when the SDK/policy requires it for downgrade or
  removal.

The no-label state is also dynamic:

- Before sign-in: do not list tenant labels or document capabilities.
- After sign-in: "No sensitivity labels are available to this account or
  tenant policy" is the normal tenant-policy result, not a sign-in retry
  instruction.
- After sign-in with a labeled document: show label/owner/protection/capability
  information rather than the no-label message.

The UI, engine, CLI/MCP, and automation surfaces must use the same
SDK-confirmed account, label, owner, policy, and rights snapshot. No UI-only
capability may bypass a provider or engine gate.

Required operation states:

- Protected document recognition.
- Sign-in prompt/account hint.
- Sign-in progress, success, cancel, timeout, retry.
- MFA/Conditional Access handoff.
- Label/current-label/label-policy display.
- View/edit/save/print/copy/comment/export/change-security rights.
- No access.
- SDK unavailable/not configured.
- Mandatory/default/downgrade-label state.
- Remove-label/remove-protection/export-required/justification state.
- Ordinary password and MIP effective rights.

UI, engine, CLI, and automation/MCP must use the SDK-confirmed rights and must
not expose independent bypass paths.

## Security and privacy standard

- Use the official SDK for decryption and authorization.
- Never decrypt or unprotect independently to implement a bypass.
- Stage original and decrypted files with owner-only/private permissions.
- Bound temporary file sizes and allocations.
- Delete temporary plaintext and SDK output promptly after use.
- Never log access tokens, passwords, use-license content, or private tenant
  data.
- Use no application-only SuperUser role for public desktop sign-in.
- Never mark package metadata preservation as live MIP support.
- Treat unknown or unreadable rights as denied.
- Separate ordinary password security from MIP rights and intersect them
  rather than choosing the weaker or stronger layer incorrectly.
- Preserve unknown package and document parts without misrepresenting their
  security state.

## Ownership and review

Recommended owners:

- **Legal/business**: IPIA request, contract, new app IDs, and trust boundary.
- **Storytold Entra/security**: app registration, permissions, consent, app
  ID, identity, certificate custody, and security review.
- **Release engineering**: official SDK package, helper build, OS dependencies,
  signing, notarization, package, and evidence.
- **PdfCraft/app maintainers**: format adapter, rights/UI/automation, tests,
  and tenant POC.
- **Customer Purview administrators**: customer license, labels, policy,
  consent, users, recipients, and revocation.
- **Support**: incident runbook, known failures, tenant escalation, and
  customer communication.

No team should mark a public MIP feature complete while another owner still
has an unresolved prerequisite.

## Evidence standard

Every public or live-MIP completion record must include:

- SDK package name/version/SHA-256/source.
- App ID and IPIA/change record; do not publish tokens or secrets.
- App/platform/package build ID.
- Signing/notarization evidence.
- Tenant test environment without personal data.
- PFILE or protected PDF fixture type and protection level.
- Test matrix and exact pass/fail/skip counts.
- Sign-in/MFA/Conditional Access result.
- Use-license and rights result.
- Save/re-publish/revert/remove operations.
- Password/MIP intersection result.
- UI/CLI/MCP/automation evidence.
- Remaining unsupported or blocked cases.

The POC PR and RFC issue are linked public evidence. Private credentials,
tokens, tenant IDs that are sensitive, and customer personal data must remain
outside source and public issue/PR comments.

## Acceptance criteria

RFC-0041 is implemented for the suite standard when:

1. One suite app ID and one IPIA scope are recorded.
2. The exact SDK package and redist inventory are recorded.
3. Each public app/platform package is signed/notarized and helper discovery is
   tested.
4. The customer-tenant onboarding procedure is documented and repeatable.
5. Every app claiming live MIP support has a real format adapter and tenant
   POC.
6. Metadata pass-through is not described as live MIP support.
7. Ordinary password protection remains independent and can intersect with MIP
   rights safely.
8. UI, engine, CLI, MCP, save/export/print/edit/copy rights use the same SDK
   decisions.
9. Temporary decrypted/working files are private and cleaned.
10. Recurrence, renewal, evidence, and failure owners are assigned.

## Official references

- [Microsoft MIP SDK setup and configuration](https://learn.microsoft.com/en-us/information-protection/develop/setup-configure-mip)
- [Microsoft MIP SDK API permissions](https://learn.microsoft.com/en-us/information-protection/develop/concept-api-permissions)
- [Microsoft MIP SDK supported file types](https://learn.microsoft.com/en-us/information-protection/develop/concept-supported-filetypes)
- [Microsoft MIP SDK overview](https://learn.microsoft.com/en-us/information-protection/develop/overview)
- [Microsoft Entra app registration quickstart](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app)
- [Configure desktop apps that call web APIs](https://learn.microsoft.com/en-us/entra/identity-platform/scenario-desktop-app-configuration)
- [SharePoint-compatible IRM PDF readers](https://learn.microsoft.com/en-us/purview/sp-compatible-pdf-readers-for-irm)
- [Microsoft MIP SDK binaries](https://aka.ms/mipsdkbinaries)
