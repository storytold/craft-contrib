# RFC-0015: Confined file access for agents

Related: RFC-0009 (shared infrastructure, "security boundaries"), RFC-0012
(command-line conventions) and RFC-0004 (duplicate PRs), all still proposals; storytold/effectcraft#2131 (audit F-11).

## Summary

Every Craft CLI that runs an agent (`<app>-cli mcp`, and any other headless
automation entry point) accepts `--automation-read-root DIR` and
`--automation-write-root DIR`. The flags mean the same thing in every app,
and every app passes one shared set of escape tests. The path checks live in
one shared crate in `craft-libs` instead of a copy per app.

## Motivation

Agents now drive Craft apps on people's own computers and on shared servers,
where one process works for one person and must not touch anyone else's files.
The folder lock is what makes that safe, and today each app has its own:

- PhotoCraft, VectorCraft, WordCraft, DeckCraft, PdfCraft, DesignCraft,
  FigCraft, SoundCraft and EffectCraft (from 0.19.0) ship the flags.
- CADCraft, FilmCraft, GridCraft and LightCraft don't yet. Each has at least
  one open PR adding them, and GridCraft has three
  (cadcraft#1904, filmcraft#1657, gridcraft#480, #2739 and #2857,
  lightcraft#2176).
- AniCraft has no CLI yet.

The copies already disagree, and the gaps show up in the same places:

- **Relative paths.** EffectCraft refuses them (effectcraft#2478); DesignCraft,
  FigCraft and SoundCraft resolve them inside the root.
- **Paths inside documents.** A document opened from inside a read root can
  name files outside it: footage, linked images, fonts, included files. The
  command checks don't see those paths. EffectCraft 0.19.0 still renders
  outside footage this way (effectcraft#2131 F-11; a fix is proposed in
  effectcraft#2719).
- **Flags that do nothing.** A FilmCraft build once accepted the flags and
  ignored them, so `--help` alone doesn't prove a build is confined. RFC-0012's
  proposed "reject unknown flags" rule would cover the parsing. This RFC covers what the
  flags must do.
- **Code and network.** Each app decides on its own whether scripts, plug-ins,
  running programs, URL fetches and control ports are allowed while confined.
- **Settings that point elsewhere.** SoundCraft 0.14.0 accepts
  `setup.disk_allocation` with a record folder outside the roots
  (soundcraft#1224). Nothing records into that folder yet, but it would once
  recording uses the setting; soundcraft#1225 refuses the outside path.

## Proposal

### 1. What the flags mean

1. `--automation-read-root DIR` and `--automation-write-root DIR` are each
   repeatable. A write root can also be read.
2. With either flag present, the process is **confined**. Without them, nothing
   changes.
3. A root that doesn't exist, or that isn't a directory, is an error at
   startup, before anything else runs.
4. Every path is checked after resolving it: `..` collapsed, symbolic links
   followed, and the result compared with the resolved roots. A path that
   can't be resolved, such as a new file in a folder that doesn't exist, is
   checked through its nearest existing parent.
5. A relative path is resolved against the first write root, or the first
   read root when there's no write root. (This replaces EffectCraft's
   "absolute paths only" rule.)
6. A refusal is an error that names the path, the access (read or write) and
   the roots. It never falls back to a default location.

### 2. What is checked

The check sits where files are opened, not only where commands take
parameters:

1. Every path a command is given.
2. Every path a command writes to without being given one: Save, autosave,
   recovery files, render and export queues, caches, settings and logs. Each
   must go inside a write root, go to a folder the process was given for its
   own use, or be turned off while confined.
3. Every path a document names: linked or placed media, footage, fonts,
   includes, external references. A path outside the read roots is treated
   as missing, so the document opens with that item offline, and the reply
   says why.
4. Folder listings, existence checks and file metadata, since those tell an
   agent what is outside the roots too.

### 3. What is refused while confined

- Running scripts, macros or expressions that can reach files, processes or
  the network.
- Running other programs, opening other applications, revealing files in the
  system file manager.
- Loading plug-ins (CLAP, VST, effect, or font plug-ins) from any path.
- Fetching URLs, unless the app has an allow-list that is empty by default.
- Opening listening sockets, such as a control port.
- Granting scripts the file or network access the user withheld.

The refusal says which rule applied, for example "`script.run` is not
available while file access is confined".

### 4. One shared crate

Under RFC-0009's "security boundaries" exception, `craft-libs` gets a
`confine` crate with:

- `Confinement { read, write }`, built from the parsed flags;
- `check(path, Access) -> Result<PathBuf, Refusal>`, with the resolution rules
  in section 1;
- a file-access wrapper an app's engine can read through, so that section 2's
  item 3 doesn't depend on knowing which command led to the read;
- the shared escape tests (section 5) as a test-support module.

Apps keep their own lists of commands to refuse. Those depend on each app's
command catalogue. The crate's maintainer is named in the PR that adds it.

### 5. Shared escape tests

Each app runs the same cases in CI, against a temporary folder used as the
root, with a second folder outside it holding a marker file:

| Case | Paths tried |
|---|---|
| absolute outside | `/etc/passwd`, `<outside>/secret.*` |
| `..` | `../secret.*`, `<root>/../secret.*` |
| symlinked file | `<root>/link.*` → `<outside>/secret.*` |
| symlinked folder | `<root>/dir/secret.*`, where `dir` → `<outside>` |
| write through a link | saving over `<root>/link.*` must not change the target |
| generic command tools | the same paths through `execute`, `run_command`, `batch` |
| paths inside a document | a document in the root naming each path above |
| refused features | one call per item in section 3 |

A case passes when the call is refused with a path or confinement error and
the outside folder is unchanged. Afterwards, the test also checks that no new
listening ports were opened and no files were created outside the roots.

We have run most of these cases against the release builds of the nine apps
that ship the flags, using a Python harness inside a container with no network
and a read-only filesystem. Paths inside documents were tested where the format
links files: WordCraft, DeckCraft, DesignCraft, VectorCraft, SoundCraft and
EffectCraft. The refused-features row was checked per app, not as one shared
list. We will port the harness to Rust for the crate and fill those gaps there.

## Rollout

1. The `confine` crate lands in `craft-libs`, with the tests.
2. Apps that already ship the flags move to it one PR at a time. Each PR passes
   the shared tests before and after the move, so behaviour only changes where
   an app had a gap.
3. For CADCraft, FilmCraft, GridCraft and LightCraft, the open PRs adopt the
   crate instead of their own checks. As RFC-0004 proposes, one PR per app goes
   ahead and the others are closed in its favour.
4. An app's README says "follows RFC-0015" once it passes.

## Alternatives

- **Keep a copy per app.** That's how things stand, and it has already led to
  different rules and the same gaps found more than once.
- **Rely on the operating system** (containers, `landlock`, App Sandbox). This
  is worth doing as well, but it doesn't help someone running an app on their
  own laptop. It's also different on every platform, and an error from the OS
  is less useful to an agent than one that names the root.
- **Check only command parameters.** This misses paths inside documents,
  which is the hole effectcraft#2131 F-11 describes.

## Open questions

- Should a confined process also get a per-session temporary folder flag
  (`--automation-temp-dir`), so caches and recovery files have a place that
  isn't a user's root?
- Should the bridge mode (`mcp --connect`/`--bridge`, which drives the running
  desktop app) refuse to start with the flags, since the desktop app's session
  isn't confined?
