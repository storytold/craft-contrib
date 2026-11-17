# RFC-0041: One plug-in runtime and one way to install plug-ins

Related: RFC-0040 (shared library first), RFC-0015 (confined file access),
storytold/photocraft#662 (a capability model for plug-ins).

## Summary

Four Craft apps run WebAssembly plug-ins today, each with its own sandbox,
manifest, permission model and way of installing them. Ten apps have none. This
RFC proposes one shared crate in `craft-libs`, `craft-plugin`, that holds the
parts every app needs: the sandbox and its limits, the manifest's common fields,
permissions checked against the app's command registry, the bundle format, the
per-user plug-in folder, and installing from GitHub releases through a catalog
or straight from a repository. Each app keeps what is its own: the plug-in kinds
that touch its document model (pixel filters, live effects, effects, widgets),
its list of refused commands, and its menus.

The result for people who use the apps: a plug-in is installed the same way in
every Craft app, from a GitHub repository, without anyone opening a pull request
against the app. The result for people who write plug-ins: one manifest, one
ABI for command plug-ins, one publishing guide, and a module that can declare
that it works in several apps.

## Motivation

### Four implementations already exist

| App | Crate | Size | What a plug-in can do | How it is installed |
|---|---|---|---|---|
| FigCraft | `crates/plugin` (`figcraft-plugin`) | ~2,900 lines | Call any allowed command, show a panel, keep settings, draw widgets, offer a file to save (ABI 6) | Import a manifest, or a team's own `figcraft-server` directory with SHA-256 checks |
| PhotoCraft | `crates/plugins` | ~2,100 lines | Pixel filters on the active layer (ABI 1) | `plugin.install` from a path or bytes, or a preference folder |
| VectorCraft | `crates/plugins` | ~12,300 lines | Object filters, live effects, exporters, importers, command calls with permissions (ABI 1, API level 2) | `plugin.install`, a preference folder |
| EffectCraft | `crates/plugin` (`effectcraft-plugin`) | ~1,000 lines | Effects in the render pipeline (WebAssembly ABI 1 and 2) | The `Plug-ins` folder, and a Plug-in Library that reads a catalog (`index.json` in `storytold/effectcraft-plugins`) and downloads GitHub release assets pinned by SHA-256 |

All four chose the same foundation independently: `wasmi` 2 (a pure-Rust
interpreter, no JIT, so the same module runs in the browser build), no WASI,
fuel and memory limits, one undo step per run, and a JSON manifest. They differ
in everything around it: manifest field names, permission names
(`permissions.access` and `permissions.commands` in FigCraft; `doc.read`,
`doc.write`, `state` and kind-specific names in VectorCraft), where plug-ins are
kept, and how they are found and updated.

Line counts are `*.rs` files in each crate at the time of writing, tests
included, rounded.

### Ten apps have none

DesignCraft, TextureCraft, CADCraft, LightCraft, FilmCraft, SoundCraft,
WordCraft, GridCraft, DeckCraft and PdfCraft have no WebAssembly plug-ins.
SoundCraft hosts native audio plug-ins (CLAP, AU), which are a different thing
and out of scope here. Several roadmaps list the gap and weight it: DesignCraft
and PhotoCraft give "Ecosystem / plug-ins / scripting" 15 % of their full target,
and WordCraft's "Ecosystem and plugins" (15 %) waits on "owner decisions". This
RFC is meant to be that decision for the parts the apps share.

### Copying a fifth time is what RFC-0040 asks us not to do

Each of those apps could copy FigCraft's crate. That would make fourteen copies
of a security boundary, each fixed separately. RFC-0040 asks contributors to
check `craft-libs` first and extend the shared API instead; there is nothing
there to extend yet.

### Adoption

The apps aim to be alternatives to large commercial suites. Those suites are
chosen partly for their ecosystems. A plug-in model where anyone can publish
from their own GitHub repository, and where the same module works in several
Craft apps, lets the ecosystem grow without the maintainers reviewing every
extension as a pull request against the app.

## Proposal

### 1. What is shared and what stays in the app

Shared, in `craft-plugin`:

- the sandbox: module checks, fuel, memory cap, host-call budget, log limits,
  string limits, and the failure model (a trap, panic, refusal or exhausted
  budget fails the run and the app rolls back);
- the manifest's common fields and their validation (section 2);
- the **command plug-in** kind and its ABI (section 3);
- permissions checked against a command registry (section 4);
- the bundle format, its SHA-256, and version comparison;
- the per-user plug-in store (section 5);
- the catalog format, installing from GitHub, and update checks (section 6).

Stays in each app:

- plug-in kinds that read or write the app's own model directly: PhotoCraft's
  pixel filters, VectorCraft's live effects, exporters and importers,
  EffectCraft's effects, FigCraft's widgets. These keep their ABIs. They use the
  shared sandbox, manifest, store and installation where that fits, and declare
  their kind in the manifest (section 2);
- the list of commands no plug-in may call (`file.*`, `app.*` and the like), and
  any app-specific refusal rules (FigCraft's widget-data rule, for example);
- running a plug-in as one undo step, and rolling back on failure;
- menus, dialogs and the Plug-ins window, drawn with `craft-ui` and `craft-menu`.

### 2. One manifest: `craft-plugin.json`

The common fields follow FigCraft's manifest, which already covers the most
cases, extended with `apps` and `kind`:

```json
{
  "id": "io.github.example.palette-tools",
  "name": "Palette tools",
  "version": "1.2.0",
  "author": "Example",
  "description": "Makes swatches and palettes from a list of colours.",
  "license": "MIT OR Apache-2.0",
  "repo": "https://github.com/example/palette-tools",
  "main": "palette_tools.wasm",
  "kind": "command",
  "apps": {
    "figcraft": { "api": 6 },
    "designcraft": { "api": 6 },
    "vectorcraft": { "api": 6, "minApp": "0.30.0" }
  },
  "permissions": {
    "access": "write",
    "commands": ["select.*", "swatch.*"]
  },
  "commands": [
    { "id": "grid", "label": "Swatch grid…", "params": [] }
  ]
}
```

- `kind` is `command` (the shared kind) or an app's own kind (`filter`,
  `effect`, `widget`, …). An app ignores a plug-in whose kind it does not know,
  and says so.
- `apps` lists the apps the module was written for, the shared command ABI
  version each needs, and optionally the oldest app version. An app refuses a
  plug-in that does not name it. Command ids differ between apps, so a module
  that works in several apps is one that was written and tested for each.
- Parameter types are FigCraft's (`text`, `number`, `boolean`, `select`,
  `list`, `slider`, `color`), so one generated dialog serves every app.
- Existing manifests (`figcraft-plugin.json`, PhotoCraft's, VectorCraft's and
  EffectCraft's embedded manifests) keep working. The apps read them as they do
  today and map them to the shared fields; nothing already published breaks.

### 3. The command plug-in kind

A command plug-in acts on the document only through the app's commands, the
same ones the menus, the CLI, the control channel and the MCP server run. Every
Craft app already has such a registry (`crates/engine/src/cmd.rs` or
`commands.rs` in each repository), which is why this kind can work in every
app. Its ABI is FigCraft's current ABI, and keeps its numbering (6 today): the host
import module and the entry point are renamed from `figcraft` to `craft`
(`craft_run`), and the old names stay accepted, so FigCraft plug-ins already
published keep working.
Panels and settings are part of it. Widgets stay FigCraft's.

The app implements one trait, the one FigCraft already has
(`figcraft_plugin::Host`): run a command with JSON parameters, return the
selection, notify the user, log, read and write the plug-in's settings, and
offer a file to save. The trait has no FigCraft-specific types. "The selection"
is whatever the app's selection is, as JSON; an app with none returns `[]`.

### 4. Permissions

A manifest asks for `access` (`read` or `write`) and may narrow `commands` to an
allowlist (exact ids, or prefixes ending in `*`). The shared crate checks the
allowlist and the app's refused list before every call. For `read`, the
registries differ in what they record (FigCraft's `history`, WordCraft's
`mutates`, DesignCraft's `journal`, none in some apps), so the adapter answers
"did this call change the document" after the call, from the app's undo history
or document revision, and a change is undone and refused; an app whose registry
knows a command edits may refuse it before the call too, as FigCraft does. Whatever the manifest says, a plug-in never gets a path,
the disk, the network, the clock, other documents, the undo history, or another
plug-in. VectorCraft's and PhotoCraft's capability names (#662) map onto these;
an app may add capability names for its own kinds.

### 5. Where plug-ins are kept

Per user, in `Plug-ins/` in the app's settings folder as `craft-settings`
reports it (or beside a portable install), one folder per plug-in named by a
hash of its id, holding the bundle and a record of where it came from (file,
catalog URL, or repository and release tag) and its SHA-256. The CLI and the
MCP server install for the session only, as FigCraft's do. The web builds keep
plug-ins in browser storage.

### 6. Installing from GitHub

Three sources, all ending in the same checks:

1. **A file**: a `.craftplugin` bundle or a manifest on disk (development).
2. **A catalog**: an `index.json` at an `https://` or `file://` URL, in
   EffectCraft's catalog format generalized with `kind` and `apps`. Each entry
   pins a GitHub release asset by URL, SHA-256 and size. The default catalog is
   one community catalog for the suite (section 8 asks where it lives); a
   person can add others, such as a company's own.
3. **A repository**: a person types `owner/repo` (optionally `@tag`). The app
   reads the repository's latest release through the GitHub API with the HTTP
   client `craft-update` already has (`parse_github_release`, proxies,
   `is_newer`), picks the `.craftplugin` asset, and reads its SHA-256 from a
   `SHA256SUMS` asset in the same release.

Checks for every source, before anything is written: `https://` only, GitHub's
hosts only (EffectCraft's rule: `github.com`, `raw.githubusercontent.com` and
the hosts release assets redirect to), size limits, the SHA-256, the manifest,
the API version, the `apps` entry, and the module compiled but not run.

**Consent.** Install opens a confirmation that lists the permissions, the kind,
the source and the SHA-256. A plug-in from a repository that is in no catalog
says so plainly ("not in any catalog you use"). Nothing is written before the
person presses Install. From a repository, the permissions are in the bundle,
so the bundle is downloaded into memory and checked first; a release must carry
a `SHA256SUMS` entry or GitHub's asset digest for it, and exactly one
`.craftplugin` asset. An update repeats the confirmation when permissions
change.

**Updates.** Browsing the Plug-ins window, or `plugin.update`, is the update
check. Nothing checks in the background unless the person turns it on.

**Publishing.** A template repository (a Rust plug-in, a `rust-toolchain.toml`,
a workflow that builds the module reproducibly from a tag and attaches the
bundle and `SHA256SUMS` to the release) makes publishing a matter of pushing a
tag. Listing in a catalog is a pull request to the catalog's repository, not to
any app.

### 7. Commands

Every app offers the same `plugin.*` commands, so the CLI, the control channel,
MCP and scripts work alike: `plugin.list`, `plugin.install` (`path`, `data`,
`repo`, or `catalog` + `id`), `plugin.update`, `plugin.remove`, `plugin.run`,
`plugin.catalog.list` and `plugin.catalog.add`. Apps with existing `plugin.*`
commands keep their parameters and add the new ones.

## Rollout

1. `craft-plugin` lands in `craft-libs`, extracted from `figcraft-plugin` with
   its tests, plus the store, catalog and GitHub installation, reusing
   `craft-update` and `craft-settings`. It follows the shared Rust rules: no
   `unsafe`, the crate-root `deny` attribute, hostile input, bounded
   allocations.
2. FigCraft moves to it with no change in behaviour: its existing tests, and
   plug-ins built for ABI 6, still pass.
3. One app without plug-ins adopts command plug-ins (DesignCraft is proposed:
   it has a command registry with documented parameters and a scripting
   surface), to show the adapter is small.
4. The remaining apps adopt command plug-ins, one pull request each, after this
   RFC is accepted. PhotoCraft, VectorCraft and EffectCraft move their store and
   installation to the shared crate and keep their own kinds.
5. Strings for the Plug-ins window and the install confirmation go into
   `craft-i18n`'s shared catalog.

Steps 1 to 3 are offered as draft pull requests linked from this RFC, as a
reference implementation. None of it is normative until this RFC is accepted.

## Alternatives

- **Native plug-ins** (`.dll`, `.so`, `.dylib`): Rust has no stable ABI, they
  need `unsafe`, they cannot run in the browser builds, and a plug-in from a
  stranger's repository would have the person's whole account. Rejected, as the
  four apps already rejected it.
- **A JavaScript engine**: the house rules keep handwritten JavaScript out of
  native apps, and it would be a second sandbox to secure. EffectCraft's
  ExtendScript-compatible scripting is a compatibility feature and stays as it
  is.
- **A JIT runtime** (`wasmtime`, `wasmer`): faster, but not available in the
  browser builds, and larger. The apps chose `wasmi`; a later RFC can add a JIT
  back end behind the same interface for native builds if heavy plug-ins need it.
- **Keep one implementation per app**: what we have. It means fourteen security
  boundaries and fourteen ways to publish.
- **Pull requests to each app's repository**: what the catalog and repository
  sources replace for extensions that don't need to be in the app.

## Open questions

1. Where the community catalog lives: one `storytold/craft-plugins` repository
   for the suite, or one per app as EffectCraft planned
   (`storytold/effectcraft-plugins`). This RFC prefers one, filtered by `apps`.
2. Whether to verify GitHub artifact attestations (build provenance) in
   addition to SHA-256, and whether a catalog may require them.
3. Whether installing straight from a repository that is in no catalog should
   be on by default or behind a setting.
4. ArtCraft's desktop app has a web frontend rather than an egui one; how it
   would host plug-ins is left to a later RFC.
5. Catalog access from the web builds (cross-origin requests to GitHub's hosts).

## Decision process

This affects several apps and adds a shared crate, so it follows RFC-0000 with
the three-day comment period.
