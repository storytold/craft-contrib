# RFC-0082: Administrator policies for the launcher and the Craft apps

## Summary

An organization can set the ArtCraft Launcher's and each Craft app's settings,
and lock them, through Intune, Group Policy, a macOS configuration profile or a
Linux system file. Every app reads its own policy through one craft-libs crate,
`craft-policy`. The suite ships one ADMX/ADML template for all of its apps.

Affects: craft-libs (new crate), craft-launcher (templates, admin guide,
install and update policy), and every app in the suite (PhotoCraft,
VectorCraft, FilmCraft, LightCraft, PdfCraft, EffectCraft, DesignCraft,
FigCraft, WordCraft, SoundCraft, DeckCraft, CADCraft, GridCraft) as each one
adopts it.

## Motivation

Nothing in the suite reads administrator policy yet. An organization that
deploys the apps can't turn off the update check
([craft-libs#769](https://github.com/storytold/craft-libs/issues/769)), pin a
version for its fleet, or stop users changing a setting it has decided.
[RFC-0025](0025-suite-accounts-and-tokens.md) left enterprise policy hooks out
of scope.

[pdfcraft#5054](https://github.com/storytold/pdfcraft/pull/5054) adds a policy
reader to one app, with one policy (`DisableJavaScript`). If each app does the
same on its own, admins get thirteen key layouts, thirteen templates and
thirteen sets of rules, and Intune imports at most 20 custom ADMX files. The
convention has to be decided once, for all apps, before more of them ship one.

## Proposal

### 1. Each app reads its own policy

Each app reads its policy at launch through `craft-policy`, so an app started
from the Start menu, the Dock or a shell is managed the same as one started by
the launcher. The launcher does not push settings into apps. It reads its own
policy and enforces install, update and version policy, including the version
pins it already hands apps through `CRAFT_VERSIONS_FILE`.

### 2. Where policy lives

| Platform | Locks | Defaults |
| --- | --- | --- |
| Windows | `Software\Policies\ArtCraft\<App>`, in HKLM and HKCU | `Software\Policies\ArtCraft\<App>\Defaults` |
| macOS | managed preferences for domain `ai.storyteller.<id>`, forced values only | a forced `Defaults` dictionary in the same domain |
| Linux | `/etc/<id>/policy.json` (`/usr/local/etc/<id>/policy.json` on FreeBSD), a JSON object | a `Defaults` object in the same file |

- `<App>` is the app's name and `<id>` its id in craft-libs
  `crates/update/src/suite.rs`: `ArtCraft\PdfCraft` and
  `ai.storyteller.pdfcraft`, for example.
- The launcher uses `ArtCraft\Launcher`, `ai.storyteller.artcraft-launcher`
  and `/etc/artcraft-launcher/policy.json`.
- These names never change once released. An app that is renamed keeps its
  old key.
- The JSON file is the object #5054 reads, for example
  `{"DisableUpdateCheck": true, "Defaults": {"Theme": "dark"}}`.
  - Unknown keys are ignored, so a newer file works with an older app.
  - A file over 64 KB, or anything there that isn't a regular file, counts as
    unreadable.
  - So does a file anyone but root can write. This check is new; #5054 does
    not have it.
- On Windows, a switch is a `REG_DWORD` 0 or 1, a number a `REG_DWORD`, and a
  choice or text a `REG_SZ`.

Microsoft's ADMX ingestion for Win32 apps refuses policies that write under
`System`, `Software\Microsoft` or `Software\Policies\Microsoft`, with listed
exceptions. Its sample uses `software\policies\<vendor>\<app>`
([Win32 and Desktop Bridge app ADMX policy ingestion](https://learn.microsoft.com/en-us/windows/client-management/win32-and-centennial-app-policy-configuration)).

On macOS, #5054 reads a value only when `CFPreferencesAppValueIsForced` says
it is forced, so a value the user writes with `defaults write` is not policy.
We could not read Apple's page for that function (it needs JavaScript). The
macOS PR in craft-libs cites it before it merges.

### 3. Two layers: locks and defaults

- A **lock** sets a value the user can't change.
  - The control shows the value in force and is not editable.
  - It says in text, not colour alone, that the organization set it. When the
    `OrganizationName` policy gives a name, the text uses it.
  - The user's own value is kept, and comes back when the lock is removed.
- A **default** sets the starting value. The user can change it, and their
  change sticks.

Precedence, highest first:
1. lock;
2. the user's own changed value;
3. admin default;
4. built-in default.

This means each app records which settings the user has actually changed, not
only their current values.

### 4. Both hives: the stricter lock wins

On Windows, when HKLM and HKCU both set a lock, the stricter value applies, as
#5054 does for its switch. #5054 also records which hive the winning value
came from. For a tie, HKLM's value is reported.

How "stricter" is defined for each kind:
- **Switch:** each policy declares which state is stricter (see 7).
- **Number:** each policy declares whether lower or higher is stricter.
- **Choice:** the policy lists its values strictest first.
- **Text and list:** these have no order, so HKLM's value wins, and HKCU
  applies only when HKLM doesn't set the policy.

For defaults, HKLM wins over HKCU.

A signed-in Windows 11 user can write `HKCU\Software\Policies` without
elevation (tested on #5054). So an HKCU value can only tighten a setting, and
only HKLM is a lock the user can't undo. The admin guide says so. We found no
Microsoft page that states the permissions on this key either way.

### 5. Invalid values

- An invalid or unreadable **lock** fails closed: the strictest value applies.
  - The settings UI, automation results and the log say which policy, where it
    was read and why it was rejected.
  - A text or list policy has no strictest value. When it can't be read, the
    setting stays locked at its built-in default, with the same reason shown.
- An invalid **default** is ignored: the built-in default applies, and the
  same three places say which policy and why.
- No policy means no change in behaviour.

### 6. Automation, accessibility and wording

- Setting a locked value through the automation interface fails, with an error
  naming the policy and where it came from (AGENTS rules 14 and 15).
- Reading settings reports, for each one, four things:
  - the user's own value;
  - the value in force;
  - its source: lock, user, admin default or built-in;
  - for a lock, where it was read.
- A locked control exposes its role, its name, its value and that it is not
  editable. A lock that appears while the app is running is announced through
  a live region (AGENTS rule 21).
- Labels in the apps are each app's own wording. They go through translation
  and the naming check of [standards/naming.md](../standards/naming.md).
- **Departure from rule 16:** the ADMX/ADML titles and explanations are
  English only and don't go through the translation system, because Intune
  imports only en-US ADML files.

### 7. Each app defines its policies in a committed file

Each app commits `policy/policies.toml`, which `craft-policy` parses:

```toml
app = "pdfcraft"

[[policy]]
name = "DisableUpdateCheck"   # registry value name, and plist and JSON key
kind = "switch"               # switch | number | choice | text | list
class = "both"                # machine | user | both
stricter = "on"               # switch: "on" or "off"; number: "lower" or "higher";
                              # choice: none (values are listed strictest first)
default_allowed = true        # may also be set under Defaults
since = 1                     # template version that added it
title = "Turn off the update check"
explain = "..."
```

- A `number` adds `min` and `max`; a `choice` adds `values`.
- The app builds its lock table from this file, and a test in each app fails
  if the two disagree.

### 8. One ADMX for the whole suite

**Layout.**
- The suite ships one ADMX/ADML pair, so it takes one of Intune's 20 custom
  ADMX slots.
- A root category, `ArtCraft`, holds one category per app. Each app's defaults
  sit in a sub-category.

**Intune limits.**
- Templates are en-US only and use no combo boxes. These are two of the limits
  on Intune's import page. Each file must also stay at 1 MB or less
  ([Import custom ADMX and ADML templates](https://learn.microsoft.com/en-us/intune/intune-service/configuration/administrative-templates-import-custom)).
- We have not checked that Intune's import accepts ADMX `list` elements.
  `list` policies stay out of the templates until that is checked.

**Updates use a new namespace.**
- Each release of the templates uses a new namespace, `ArtCraft.Policies.v<N>`.
- Intune refuses a changed file under the same namespace ("the upload fails
  with a namespace error"). Its import page suggests adding a version number
  to the namespace instead.
- An admin imports the new file beside the old one, moves their profiles to
  it, and deletes the old one.
- Registry keys and value names stay the same between versions, so values
  already deployed keep working.
- `<N>` goes up only when a release adds or changes a policy.

**Generation and release.**
- craft-launcher generates the templates with an xtask command, `cargo xtask
  admx`.
- The command reads each app's `policy/policies.toml` at that app's pinned
  release tag, so the launcher depends on no app crate (AGENTS rule 9).
- Each launcher release attaches a templates zip, with a sample
  `.mobileconfig` and a sample `policy.json`.
- `docs/administrators.md` in craft-launcher is the admin guide for the whole
  suite. It covers:
  - the keys;
  - the HKCU caveat;
  - the steps for moving to a new namespace;
  - each app's policies.

**Refresh.** Apps read policy at launch. Re-reading while running is allowed
but not required. For reference:
- Group Policy refreshes every 90 minutes, with an offset of up to 30
  ([Group Policy processing](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-processing)).
- Intune devices check in about every 8 hours
  ([Troubleshoot device profiles](https://learn.microsoft.com/en-us/intune/intune-service/configuration/device-profile-troubleshoot)).

### 9. Where the code comes from

- `craft-policy` lives in storytold/craft-libs (AGENTS rule 9).
- After pdfcraft#5054 merges, its reader moves there in a move PR that keeps
  its behaviour, limits, tests and authorship.
- A second PR then adds:
  - the app, key root, macOS domain and file path as parameters;
  - the defaults layer;
  - the other value kinds;
  - the `policies.toml` parser.
- PdfCraft uses `Software\Policies\ArtCraft\PdfCraft`. #5054 uses
  `SOFTWARE\Policies\PdfCraft` today and has not shipped, so no admin has to
  migrate. Its macOS domain and Linux path already match.
- The reader comes from Microsoft's documentation and #5054, not from any
  other product's code ([standards/clean-room.md](../standards/clean-room.md)).

### 10. Where the rule is kept

After this RFC merges, a separate PR adds `standards/admin-policy.md`, a short
form of sections 2 to 7, as [standards/naming.md](../standards/naming.md) did
for RFC-0029. App PRs follow that file.

## Rollout

1. This RFC, then `standards/admin-policy.md`.
2. craft-libs:
   - move #5054's reader into `craft-policy`;
   - then add the parameters, the defaults layer and the definition-file
     parser.

   This fixes craft-libs#769 for apps that use craft-update.
3. craft-launcher:
   - settings commands for automation;
   - then `DisableUpdateCheck`, `OrganizationName`, `cargo xtask admx` and the
     admin guide.
4. craft-launcher: update, version-pin and install policies.
5. craft-launcher: the templates zip on releases. Its workflow change is its
   own PR.
6. Each app: adopt `craft-policy` and commit `policy/policies.toml`, one PR per
   app.

## Out of scope

- Document sensitivity labels and rights management
  ([#73](https://github.com/storytold/craft-contrib/pull/73),
  [#75](https://github.com/storytold/craft-contrib/pull/75)).
- A machine-wide launcher install. The launcher MSI is per-user today, and
  policy works with it as it is.
- The web build, which has no policy.

## Alternatives

- **The launcher pushes settings to apps.** Apps started without the launcher
  would be unmanaged, and the launcher doesn't know where each app keeps its
  settings.
- **One ADMX per app.** Each update touches only one app, but the suite would
  use 14 of Intune's 20 slots.
- **One ADMX with a fixed namespace.** Before every template update, admins
  would have to delete the profiles that use it.
- **HKLM only.** Simpler, but Group Policy user settings write HKCU, and #5054
  already reads both.

## Open questions

- Should values directly under `Software\Policies\ArtCraft`, such as
  `OrganizationName` or `DisableUpdateCheck`, apply to every app, with the
  app's own key winning?
