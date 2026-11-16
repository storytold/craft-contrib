# Native Apple boundary

Proposed by [RFC-0041](../rfcs/0041-native-apple-shells.md); not an adopted standard
until its decision process completes.

## Ownership

| Owner | Keep here | Do not move here |
| --- | --- | --- |
| App Rust engine/automation | Document model, authorization, command IDs, rendering, edits/history, serialization and recovery semantics | SwiftUI, egui or OS dialog dependencies |
| App's thin Rust adapter | Map supported app operations to generated bounded wire types | A second parser, renderer, permission system or copy of engine commands |
| `craft-libs` | Existing reusable formats/protocols/atomic I/O; separately approved, genuinely cross-app host primitives | PDF commands, app tool catalogues, GitHub workflow orchestration |
| Native shell | Files coordination/scopes, platform resource lifetime, views, accessibility, export/share presentation | Document mutations, permission overrides, a second undo stack |
| App packaging | App ID, capabilities, icons/metadata, app build/archive entry point, app-specific operational tests | Copied cross-app runner or signing infrastructure |
| `craft-common-actions` | Reusable toolchain/device setup, execution, evidence/artifact and later protected distribution jobs | Document behavior or one workflow clone per app |
| `craft-contrib` | Policy and acceptance evidence | Runtime libraries, certificates or CI implementation |

Use the [existing shared-library-first proposal](https://github.com/storytold/craft-contrib/pull/62)
and [existing migration proposal](https://github.com/storytold/craft-contrib/pull/56);
this document does not create a competing general de-duplication rule.
[Native-boundary evidence](https://github.com/storytold/craft-libs/pull/1909)
demonstrates optional existing JSON-RPC helpers with synthetic adapters. It is not
an instruction to convert an app's local FFI to JSON-RPC, start a server, or adopt
an unmerged library.

## Contracts to preserve

- Serialize engine access. Native asynchronous presentation must not race mutations,
  close/export or an in-flight render.
- Bound paths, dimensions, allocations and UTF-8 output before crossing the bridge.
  Never truncate inside a Unicode scalar; expose truncation explicitly.
- Surface structured failures and cancellation without success-shaped empty results.
  Do not auto-retry a mutation after an uncertain result.
- Obtain OS access before reading a document, snapshot/import through the engine,
  and release platform resources on cancellation, failure, close and backgrounding.
  A cached path is not a lasting user grant.
- Saving a private copy and finishing an OS export are different events. Do not
  silently clear unsaved-work confirmation because serialization succeeded.
- Keep scripting, network-facing automation and external code loading off unless
  a separately accepted policy and explicit user choice allow them.
- Native discovery surfaces consume one supported-tool description, reusing stable
  command identity. Do not scrape Rust source or copy the full desktop catalogue.

## Desktop-to-mobile adaptation

Prefer the corresponding desktop workflow before designing a mobile variant.
Preserve intent and safety while adapting touch targets, system text sizes, safe
areas, keyboard/pointer input, scene interruptions and compact layouts. One task
can become an inspector/sidebar on iPad and a sheet on iPhone.

Reuse original licensed artwork and compatible metadata from desktop packaging.
Generate platform derivatives; keep only genuinely different identifiers,
orientations, entitlements and OS integrations separate. Do not copy unimplemented
document handlers or localization claims. Do not reuse restricted community marks
in a fork.

## Required evidence

Apply the [approved-native accessibility mapping](accessibility.md#approved-native-apple-hosts)
without lowering RFC-0014's bar: real keyboard/accessibility focus, meaningful
native trees and announcements, and keyboard-driven XCTest evidence alongside
captures. A raster canvas or native controls alone do not prove accessibility.

Rust operation tests, hosted native bridge tests and real native operational tests
are complementary. Cover Files import/cancel/malformed recovery, render/zoom,
mutation, save/export/reopen and sharing; check landscape, largest text, dirty-close
and errors. Use synthetic, licensed fixtures.

Record source/toolchain/runtime revisions, tested device types, exact commands and
failures/skips. Add physical iPhone/iPad, lowest-supported/current OS, VoiceOver
speech/focus, memory/suspend/recovery and file-provider evidence before release
claims. Do not infer this evidence from native labels or a universal build setting.
