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
- TextureCraft ships the flags. MaterialCraft 0.1.0 has the lock under other
  names, `--read-root` and `--write-root`.
- AniCraft has no CLI yet.

The copies already disagree, and the gaps show up in the same places:

- **Relative paths.** EffectCraft refuses them (effectcraft#2478); DesignCraft,
  FigCraft and SoundCraft resolve them inside the root.
- **Paths inside documents.** A document opened from inside a read root can
  name files outside it: footage, linked images, fonts, included files. The
  command checks don't see those paths. EffectCraft 0.19.0 still renders
  outside footage this way (effectcraft#2131 F-11). effectcraft#2719, which
  refuses that read, merged on 31 October 2026. It is in 0.20.0 and later, not
  in 0.19.0. The hosted BridgeCraft image pins EffectCraft 0.21.1
  (adikshan11/bridgecraft#66), which is after that fix.
- **Flags that do nothing.** A FilmCraft build once accepted the flags and
  ignored them, so `--help` alone doesn't prove a build is confined. RFC-0012's
  proposed "reject unknown flags" rule would cover the parsing. This RFC covers what the
  flags must do.
- **Code and network.** Each app decides on its own whether scripts, plug-ins,
  running programs, URL fetches and control ports are allowed while confined.
- **Settings that point elsewhere.** SoundCraft 0.14.0 accepts
  `setup.disk_allocation` with a record folder outside the roots
  (soundcraft#1224). Nothing records into that folder yet, but it would once
  recording uses the setting. soundcraft#1225 is still open; it would refuse
  the outside path. That report is about 0.14.0. SoundCraft 0.15.0 is a later
  release, and it is not the hosted build. 0.14.0 is not the hosted build
  either. adikshan11/bridgecraft#66, merged on 5 November 2026, pins
  SoundCraft 0.17.1, and that image blocks the command while #1225 is open.

## Proposal

### 1. What the flags mean

1. `--automation-read-root DIR` and `--automation-write-root DIR` are each
   repeatable. A write root can also be read. A read root that isn't also a
   write root can't be written: no saving over a file, creating, renaming, or
   deleting inside it.
2. An agent entry point (`<app>-cli mcp`, and any other headless automation
   entry point) is confined. It refuses to start unless both a read root and
   a write root are set. An unset root is a refusal of the feature, not
   unrestricted access.
3. A desktop launch that is not an agent entry point is unchanged when
   neither flag is present. If either flag is present, that process is
   confined, and an access whose root was not set is refused.
4. A root that doesn't exist, or that isn't a directory, is an error at
   startup, before anything else runs.
5. Resolve the path first, then compare components with the resolved roots.
   `..` is collapsed. Symbolic links are followed, and so are Windows
   junctions. The comparison is of the resolved components, so a root of
   `/sandbox` does not allow `/sandbox-other`, and the string the caller
   passed is not what is compared. On a case-insensitive file system the
   comparison is case-insensitive. A differently-cased spelling of a path
   inside a root is that same path and is allowed. A different directory
   that only shares a case-folded prefix with a root is outside and is
   refused. A UNC path (`\\server\share` or
   `//server/share`) and a device path (`\\?\`, `\\.`) are resolved and then
   compared the same way; if they do not fall inside a root, they are
   refused. An 8.3 short name is resolved to the long path before the
   comparison. The short name is not trusted on its own. A path that can't
   be resolved yet, such as a new file whose parent does not exist, is
   resolved through its nearest existing parent, and the create in section 4
   still has to land inside the root.
6. A relative path is resolved against the first write root, or the first
   read root when there's no write root. (This replaces EffectCraft's
   "absolute paths only" rule.)
7. A refusal is an error that names the path, the access (read or write) and
   the roots. It never falls back to a default location.
8. Hard links are refused, because resolving a path can't reveal them. A hard
   link inside a root and a name outside it are one file, so reading or writing
   the inside name reaches the outside one. After opening a regular file, the
   confined open reads its link count from the handle (`st_nlink` from `fstat`,
   `nNumberOfLinks` from `GetFileInformationByHandle`) and refuses the file if
   the count is above one. A link count says that another name exists, not
   where, so two names that are both inside a root are refused too. A volume
   that can't report link counts can't be used as a root, and naming one is a
   startup error. A confined process can't create hard links.
   This covers links that exist when the file is opened. It doesn't cover a
   link that another process adds to an open file, so a root must not be
   writable by anyone the boundary protects against.

### 2. What is checked

The confined open sits where files are opened. A check that returns a path,
followed by a later open of that path, is not this contract: the directory
or the symlink can change in between.

1. Every path a command is given, opened or created through section 4.
2. Every path a command writes to without being given one: Save, autosave,
   recovery files, render and export queues, caches, settings and logs. Each
   must be created inside a write root, go to a folder the process was given
   for its own use, or be turned off while confined.
3. Every path a document names: linked or placed media, footage, fonts,
   includes, external references. A path outside the read roots is treated
   as missing, so the document opens with that item offline, and the reply
   says why. The outside file is not read and not modified.
4. Folder listings, existence checks and file metadata, since those tell an
   agent what is outside the roots too. Those calls use the same resolution
   and return a refusal, not a listing, for anything outside the roots.

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
`confine` crate. RFC-0009 is still a proposal. If that exception changes,
the crate still belongs in `craft-libs`, and the maintainers name the rule
that lets shared security code land. The crate has:

- `Confinement { read, write }`, built from the parsed flags. Building it
  for an agent entry point fails if either root is unset.
- `open_read(path)`, `open_write(path)` and `create(path)`, each returning
  `Result<OpenFile, Refusal>`. One call resolves, compares and opens or
  creates, and it returns a handle. The caller uses the handle. It does not
  receive a path to open later. `check(path, Access) -> PathBuf` is not the
  interface.
- The same property on every platform the app ships. On Linux, `openat2`
  with `RESOLVE_BENEATH` or `RESOLVE_IN_ROOT` is one way to stop a directory
  from being swapped during the open. macOS and Windows do that with a
  directory handle and a final component, and refuse if the handle no longer
  names a directory inside the root. The contract is that property, not the
  Linux call.
- a file-access wrapper an app's engine can read through, so that section 2's
  item 3 doesn't depend on knowing which command led to the read. The wrapper
  uses the same open.
- the shared escape tests (section 5) as a test-support module.

Apps keep their own lists of commands to refuse. Those depend on each app's
command catalogue. The crate's maintainer is named in the PR that adds it.

### 5. Shared escape tests

Each app runs the same cases in CI, with three temporary folders: a read
root, a separate write root, and a folder outside both holding a marker file.
`<root>` in the table means each of the two roots in turn:

| Case | Paths tried |
|---|---|
| absolute outside | `/etc/passwd`, `<outside>/secret.*` |
| `..` | `../secret.*`, `<root>/../secret.*` |
| symlinked file | `<root>/link.*` → `<outside>/secret.*` |
| symlinked folder | `<root>/dir/secret.*`, where `dir` → `<outside>` |
| write through a link | saving over `<root>/link.*` must not change the target |
| junction, where the platform has them | a Windows junction inside the root pointing outside |
| UNC and device path | `\\server\share\secret.*`, `//server/share/secret.*`, `\\?\` and `\\.\` forms that resolve outside |
| 8.3 name | a short name inside the root whose long path is outside |
| case fold, same directory | on a case-insensitive volume, another spelling of a file inside the root that differs only by case |
| case fold, outside | on a case-insensitive volume, a path that shares a case-folded prefix with a root but names a different directory, such as root `/sandbox` and `/sandbox-other/secret.*`, or root `/tmp/Read` and `/tmp/readme/secret.*` |
| directory swap | a directory inside the root replaced with a symlink to outside after lookup starts, including while creating a new file in that directory |
| symlink swap | a file inside the root replaced with a symlink to outside after lookup starts |
| hard link | `<root>/hard.*` is a hard link to `<outside>/secret.*`; reading it and saving over it |
| read/write separation | in the read root: read a file, save over it, create, rename and delete. In the write root: the same calls |
| generic command tools | the same paths through `execute`, `run_command`, `batch` |
| paths inside a document | a document in the root naming each refused direct-escape path above, not the same-directory case-fold spelling |
| refused features | one call per item in section 3 |

A platform that does not have junctions, UNC paths, 8.3 names, hard links or a
case-insensitive volume skips that row. A skip is not a pass.

A direct read or write (every row except "paths inside a document",
"refused features", "read/write separation", and "case fold, same directory")
passes only when the
call is refused and the outside file is not used. The marker bytes must not
appear in the reply or in any file the call wrote, and the marker file must
be unchanged. The directory-swap row includes creating a new file while the
parent is swapped: that file must not appear outside the root.

"Case fold, same directory" is a different assertion. The call is allowed.
The bytes read or written are those of the file inside the root. The marker
bytes must not appear in the reply or in any file the call wrote, and the
marker file must be unchanged. Refusing that spelling does not pass this row.

"Paths inside a document" is a different assertion. The document opens, the
outside item is offline, and the reply says why. The marker bytes must not
appear in the reply or in any file the call wrote, and the marker file must
be unchanged. Refusing to open the document does not pass this row. An open
that reads or modifies the outside file fails it.

"Read/write separation" is a different assertion. In the read root, the read
succeeds with the file's bytes. Saving, creating, renaming and deleting are
each refused, and the folder's contents and bytes are unchanged afterwards. In
the write root, every call succeeds. An implementation that lets writes reach
the read root fails this row, even if it passes every escape row.

"Refused features" passes when the call is refused with a confinement error.
It does not open or change the outside file.

Afterwards, the test also checks that no new listening ports were opened and
no files were created outside the roots.

We have run most of the earlier rows against the release builds of the nine
apps that ship the flags, using a Python harness inside a container with no
network and a read-only filesystem. Paths inside documents were tested where
the format links files: WordCraft, DeckCraft, DesignCraft, VectorCraft,
SoundCraft and EffectCraft. The refused-features row was checked per app,
not as one shared list. That harness run did not include the junction, UNC,
8.3, case-fold (same directory or outside), directory-swap, symlink-swap,
hard-link or read/write-separation rows. A later sweep, recorded under
Decision process, did run the hard-link row and a separate read root and
write root. We will port the harness to Rust for the crate and fill the
remaining gaps there. This RFC does not claim the junction, UNC, 8.3,
case-fold, directory-swap or symlink-swap rows have been run.

### 6. Bridge mode fails closed

`mcp --connect` and `--bridge` drive a running desktop process. A confined
CLI refuses that connection unless the receiving process enforces this same
confinement: the same roots, the same open and create, and the same
refusals. Connecting a confined CLI to an unrestricted desktop process
bypasses the boundary. An app that does that is not following RFC-0015, and
this is not optional for an app that says it does.

## Rollout

1. The `confine` crate lands in `craft-libs`, with the tests.
2. Apps that already ship the flags move to it one PR at a time. Each PR passes
   the shared tests before and after the move, so behaviour only changes where
   an app had a gap.
3. For CADCraft, FilmCraft, GridCraft and LightCraft, the open PRs adopt the
   crate instead of their own checks. As RFC-0004 proposes, one PR per app goes
   ahead and the others are closed in its favour. RFC-0004 is still a
   proposal (#5). If it changes, this step is still one shared crate, and the
   maintainers choose which pull request proceeds.
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

## Decision process

A Storytold maintainer comments the RFC-0000 comment-period end date on this
pull request. This RFC does not set that date. On 6 November 2026
@Jeremy8776 commented that the period ends on Monday 9 November 2026, at the
end of the day UK time. RFC-0000 says a member of `craft-app-team` starts
the period. GitHub does not show that comment's author as a member of the
organization on this pull request, so this text does not count the comment
as the start.

craft-contrib's workflow is the shared SonarCloud wrapper. On a fork pull
request the scan step is skipped when the token is absent, and instrumented
tests are skipped when the repository has no Rust crate. A completed wrapper
check is not security validation. This text does not claim a live exploit
was reproduced. There is no `confine` crate in `craft-libs`, and the
junction, UNC, 8.3, case-fold (same directory or outside), directory-swap,
and symlink-swap rows have not been run.

The other rows have been run from outside, on Linux, against the releases
adikshan11/bridgecraft#57 pinned: WordCraft 0.15.1, DeckCraft 0.15.0,
PdfCraft 0.18.1, PhotoCraft 0.20.1, VectorCraft 0.21.0, DesignCraft 0.17.0,
FigCraft 0.15.0, SoundCraft 0.17.0, EffectCraft 0.21.0, TextureCraft 0.3.0
and MaterialCraft 0.1.0. The harness tries absolute, `..`, file-symlink,
folder-symlink, hard-link, sibling-prefix and write-into-read-root paths
through every tool and engine command that takes a path. It is
[`scripts/escape_checks.py`](https://github.com/adikshan11/bridgecraft/blob/ed06a71e42d16b2a6a4e840151877143b84688f8/scripts/escape_checks.py).
About 24,000 attempts found one problem: SoundCraft 0.17.0's
`file.export_midi` and `track.save_preset` write through a hard link already
inside the root and change the file it points to
([soundcraft#1359](https://github.com/storytold/soundcraft/issues/1359)).
[soundcraft#1360](https://github.com/storytold/soundcraft/pull/1360) would
write to a temporary file and rename it over the destination. That pull
request is still open, and the change is not in SoundCraft 0.17.1.
adikshan11/bridgecraft#66, merged on 5 November 2026, pins WordCraft 0.15.2,
DeckCraft 0.15.1, PdfCraft 0.18.2, PhotoCraft 0.20.2, VectorCraft 0.21.1,
DesignCraft 0.17.2, FigCraft 0.15.1, SoundCraft 0.17.1, EffectCraft 0.21.1,
TextureCraft 0.3.2 and MaterialCraft 0.1.2. Each app's note on that commit
says the same harness reached nothing outside the roots. SoundCraft's note
is 693 attempts. The image still blocks `file.export_midi`,
`track.save_preset` and `setup.disk_allocation`, so that run did not exercise
the two commands #1359 is about. The harness doesn't cover paths named
inside a document, or expressions in EffectCraft.
