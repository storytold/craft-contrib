# RFC-0018: Command-line conventions

## Summary

Every Craft app's command-line tool (`<app>-cli`) behaves the same way and is
built the same way: [clap](https://docs.rs/clap) derive types that parse the
command line into typed values, [thiserror](https://docs.rs/thiserror) errors,
the same exit codes and output streams, and generated shell completions and man
pages. A CLI that follows these rules says so ("follows RFC-0018"), so a user
who knows one Craft CLI knows them all, and a reviewer can check a CLI against
one list instead of re-arguing each choice per app.

The rules come from photocraft's move to clap
([photocraft#7071](https://github.com/storytold/photocraft/issues/7071), merged
in [#9167](https://github.com/storytold/photocraft/pull/9167) and
[#9194](https://github.com/storytold/photocraft/pull/9194)), which is the
reference implementation. cadcraft and soundcraft have since taken most of its pattern.

## Motivation

On 2026-11-26 the `main` branch of the storytold repositories had 19 `<app>-cli` crates: anicraft,
cadcraft, chatcraft, craft-anything, deckcraft, designcraft, effectcraft, figcraft, filmcraft,
gridcraft, lightcraft, loopcraft, materialcraft, pdfcraft, photocraft, soundcraft, texturecraft,
vectorcraft and wordcraft. Seventeen are in a release (chatcraft and loopcraft are not). Appendix A
lists them. Reading their source and running the released Linux x86_64 binaries shows:

- **Five use clap, fourteen parse by hand.** cadcraft, craft-anything, figcraft, photocraft and
  soundcraft use clap derive. The other fourteen each parse their own way, in 284 to 4467 lines per CLI.
  Four crates (cadcraft, figcraft, photocraft, soundcraft) use thiserror. Fifteen of the nineteen
  return `Result<_, String>` somewhere. Only cadcraft, chatcraft, photocraft and soundcraft run through a
  `run(args, out, err)`-style function that tests can call.
- **Exit codes differ.** All 17 released CLIs refuse an unknown top-level flag and print nothing on
  stdout, but 8 exit 2 (rule 1.2) and 9 exit 1. helenkwok measured the same split on the macOS
  releases of 13 November
  ([comment](https://github.com/storytold/craft-contrib/pull/18#issuecomment-6870457576)): 8 on 2
  and 10 on 1, with `solidcraft` as the eighteenth CLI, which has no repository to read. Her
  measurement and this one are top-level behaviour; they are dated and cover one release each.
- **Flags after a subcommand are not checked everywhere.** Given an unknown flag, the `commands`
  subcommand of deckcraft, designcraft, filmcraft, lightcraft and vectorcraft exits 0 and prints its
  normal output, so a mistyped flag (`--fromat`) is silently ignored. The other subcommands
  were not probed.
- **`--version` is top-level only for 16 of 17.** Only filmcraft prints the version for
  `commands --version`. pdfcraft does not accept `-V`. Eight CLIs accept a `version` subcommand
  (anicraft, deckcraft, designcraft, effectcraft, filmcraft, gridcraft, lightcraft, texturecraft).
- **Overwriting is a flag in the clap CLIs only.** `--in-place` exists in cadcraft, photocraft and
  soundcraft. No CLI reads an environment variable into a destructive flag today. The variables they
  read are tokens, keys, paths, ports, locale and logging.
- **Completions and man pages ship from three apps.** The Linux tarballs of cadcraft, photocraft and
  soundcraft install bash, zsh and fish completions and a man page per command. The other fourteen
  released apps ship none, and no macOS CLI zip carries them (helenkwok's finding, checked on
  photocraft's zip).

The problems photocraft had before its port show up in the hand-written CLIs:

- A mistyped flag (`--fromat`) was silently ignored, and a repeated one quietly
  took the last value. The ignored flag is measured above; repeated flags were not.
- A bad value failed late, after work had started, or not at all.
- Errors were strings, so a caller couldn't tell a usage error from a failure,
  and messages drifted from flag to flag.
- `--help` was hand-written text that drifted from what the parser accepts.

Fixing these per app, with different answers each time, would give users nineteen
different CLIs and reviewers nineteen different sets of rules to remember.
Photocraft has already answered the questions, through review by several
maintainers, and cadcraft and soundcraft have taken most of its pattern, so the
other apps can follow it.

## Proposal

### 1. What a Craft CLI looks like

1. **Name and shape.** The binary is `<app>-cli`, with subcommands
   (`<app>-cli convert …`). `--help`/`-h` and `--version`/`-V` work everywhere;
   `help <subcommand>` works too. There is no `version` subcommand.
2. **Exit codes.** `0` success, `1` the command failed, `2` usage error. When a signal ends the
   process, the shell's `128 + signal` (130 for Ctrl-C). A command adds no other code.
3. **Streams.** Results go to stdout: JSON when a program is meant to read them
   (pretty-printed by default, one line with `--compact` or as the command
   documents). Errors, warnings and progress go to stderr, errors as
   `error: <message>`. A subcommand that speaks a protocol on stdio (an MCP or
   JSON-lines server) writes nothing else to stdout. Where a subcommand has
   `--json`, it means the same everywhere: JSON alone on stdout, nothing else
   on that stream, and the exit code unchanged. A subcommand needs `--json` only
   where a program is expected to read its output.
4. **Strict parsing, before any work starts.** These are usage errors (exit 2,
   on stderr): a flag the subcommand doesn't take, a flag given twice unless it
   is documented as repeatable, a value of the wrong type or out of range,
   malformed JSON in a JSON-valued flag, and flags that conflict or that the
   mode doesn't use. A value flag never takes the next flag as its value: a
   value starting with `-` is written `--out=-image.png`; negative numbers are
   accepted where they mean something.
5. **Paths are paths.** Arguments are read as `OsString` and paths kept as
   `PathBuf`, so a file name that isn't valid Unicode reaches the command.
6. **Nothing destructive by default.** An output that would replace an input is
   refused unless a flag such as `--in-place` asks for it.
7. **Environment variables** are `<APP>_<NAME>` fallbacks for flags, declared
   on the flag (clap's `env =`) so `--help` lists them:
   - a flag wins over its variable;
   - an empty variable counts as unset;
   - a bad value is a usage error that names the variable, not the flag;
   - a secret's value is never shown (`hide_env_values`);
   - a variable never grants filesystem or network access on its own: such
     authority comes from flags only;
   - a variable never sets a destructive flag. `--in-place`, any other flag that
     overwrites or deletes (`--overwrite`, a force flag) and any flag like them
     come from the command line only and declare no `env =`. A stale
     `PHOTOCRAFT_IN_PLACE=1` in a CI job must not turn a conversion into an
     overwrite.

   clap's `env =` alone gives only the first of these. Photocraft adds the
   empty-means-unset and variable-naming parts with
   [`blank_env_is_unset`](https://github.com/storytold/photocraft/blob/1f7fc0f53696e54da79b95015abb37d7c22cdd1a/crates/automation/src/args.rs#L65) and
   [`name_env_vars`](https://github.com/storytold/photocraft/blob/1f7fc0f53696e54da79b95015abb37d7c22cdd1a/crates/automation/src/args.rs#L73).
8. **Help is the documentation.** The help text is the doc comments on the clap
   types, so `--help` is the source of truth. The app's book has one page for
   its CLI that links to it rather than repeating it.
9. **Shell completions and man pages** (bash, zsh, fish, elvish, PowerShell, and
   a page per subcommand) are generated from the same clap command by
   [clap_complete](https://docs.rs/clap_complete) and
   [clap_mangen](https://docs.rs/clap_mangen) in an example or build step. The
   generators are dev or build dependencies, not part of the binary.
   - **Linux and FreeBSD packages** install the bash, zsh and fish completions
     and the man pages, the formats that have a standard directory there. The
     elvish and PowerShell files are generated but not installed.
   - **macOS and Windows are out of scope** for installed completions and man
     pages. The macOS CLI is a zip with the binary and the licence files, which
     craft-launcher installs
     ([craft-launcher#1422](https://github.com/storytold/craft-launcher/issues/1422)).
     An app that wants them there puts them in its zip and changes the launcher in
     its own pull request. The CLI's book page says which platforms ship them.

### 2. How it is built

1. **Data first: invariants live in types.** The command line is parsed into
   clap derive structs whose fields already have their final types: enums for
   a fixed set of values, ranged integers, `PathBuf`, `SocketAddr`, parsed
   JSON. Relations between flags are declared with clap (`ArgGroup`,
   `conflicts_with`, `requires`). The code that does the work never sees a raw
   string it has to validate.
2. **Errors are types.** Each error a crate returns is a `thiserror` type: an
   enum for a set of failures, or a struct for a single one, with the parts of
   the message as fields (photocraft's
   [`NotAPort`](https://github.com/storytold/photocraft/blob/1f7fc0f53696e54da79b95015abb37d7c22cdd1a/crates/automation/src/args.rs#L61)). No `Result<_, String>`, no
   `Err(format!(…))`, no message kept in a `&str` constant. A usage error the
   parser can't express is still such a type, turned into a clap error by a
   method that picks its
   [`ErrorKind`](https://docs.rs/clap/latest/clap/error/enum.ErrorKind.html), so
   it prints and exits like clap's own.
3. **Named types over tuples** for anything that returns several values.
4. **`main` only wires the process.** It passes `std::env::args_os()`, stdout
   and stderr to a library function `run(args, out, err) -> exit code`, and
   exits with the result. Tests call `run` directly.
5. **Shared flags are declared once.** Flags that several binaries take live in
   a shared crate as clap `Args` structs and are flattened where used.
6. **Cargo features are for optional features**, not for switching off a
   crate's core dependency such as clap.
7. **Tests pin the interface.** Through `run`, an app's tests check at least:
   - `--help` and `--version` on stdout with exit 0, a failed command with
     exit 1, and each usage error with exit 2 and its message on stderr;
   - that a protocol subcommand (MCP, JSON lines) writes nothing else to
     stdout;
   - an unknown flag and a repeated one refused, and a documented repeatable
     flag accepted;
   - `--` before a positional that starts with `-`, and negative numbers where
     the flag takes them;
   - a path that isn't valid Unicode reaching the command;
   - a bad typed value, malformed JSON, conflicting flags and a flag the mode
     doesn't use, each refused before any file is written;
   - an output that would replace an input refused without the overwrite flag,
     and a destructive flag set only through the environment changing nothing;
   - where `--json` exists, JSON alone on stdout with the exit code unchanged;
   - for each variable: empty counts as unset, the flag wins, an error names
     the variable, a secret's value never shows in help or errors, and the
     variable alone grants no filesystem or network access;
   - and, in the packaging tests, that the Linux and FreeBSD packages install
     the bash, zsh and fish completions and a man page per subcommand.

### 3. Tools and enforcement

The tools are clap (derive), thiserror, clap_complete and clap_mangen, as
workspace dependencies.

Rules that can be checked mechanically are checked in CI with
[ast-grep](https://ast-grep.github.io/) through craft-common-actions'
[`ast-grep.yml`](https://github.com/storytold/craft-common-actions/blob/main/.github/workflows/ast-grep.yml)
reusable workflow. Each rule ships with its tests. A follow-up PR there adds the
first shared rules:

- `clap-error-message`: no string or `format!` passed to `Command::error`
  (rule 2.2).
- `string-error`: no `Result<_, String>` (rule 2.2).

Both were tried on photocraft: the first finds the two messages that #9194
replaced and passes on main; the second finds 640 existing uses, so an app
turns it on once it has none left.

An app's `AGENTS.md` asks coding agents to run the same scan before pushing.

### 4. Adopting it

An app adopts this RFC in one PR with a clear commit history, rather than a
stack of small PRs: the work is repeating what photocraft already did. The PR
moves the CLI to these rules, turns on the ast-grep workflow, and adds
"Follows [RFC-0018](https://github.com/storytold/craft-contrib/blob/main/rfcs/0018-cli-conventions.md)"
to the CLI's book page, listing any rule it doesn't follow yet and why, and
which paths the ast-grep scan covers. An app claims the line once rule 2.7's
tests pass and its packages install the generated files.

### 5. Reference implementation: known exceptions

PhotoCraft is the starting point for these conventions, not a claim that its
whole CLI already complies. At
[`1f7fc0f5`](https://github.com/storytold/photocraft/tree/1f7fc0f53696e54da79b95015abb37d7c22cdd1a):

- **Rule 1.1:** the root clap command declares a version but does not set
  `propagate_version`, so `--version`/`-V` work at the top level only.
  Full adoption needs version propagation and tests for every subcommand.
- **Rule 1.5:** `DropletArgs.file`, `.inputs` and `.out` use `String`,
  `Vec<String>` and `Option<String>`. They reject non-Unicode paths during
  parsing. Full adoption needs native path types and a path through execution
  that preserves them, with a non-Unicode-path test.

These are reference implementation gaps, not exceptions for the twelve ports.
The reference must list them when describing its compliance; a clean ast-grep
scan alone does not establish that these or the other behavior and packaging
requirements pass.

## Alternatives

- **Each app keeps its own parser and style.** No migration cost, but users
  meet a different CLI in each app, and every review re-argues the same
  choices.
- **A shared `craft-cli` crate instead of rules.** The shared helpers (empty
  variable counts as unset, naming the variable in an error, the completions
  and man page generator) may well move to craft-libs, but a crate can't make
  an app parse into types or use thiserror. The rules come first, and the crate
  can follow once a second app needs the helpers.
- **A template repository.** Nothing in storytold uses one, the apps already
  exist, and a template is copied once and then drifts. The reference
  implementation and the ast-grep rules stay current.
- **Lint with clippy alone.** Clippy has no lints for these project rules, and
  ast-grep rules are a few lines of YAML each.

## Open questions

- Should the rules move to a living document (`docs/cli-conventions.md`) as
  [RFC-0013](https://github.com/storytold/craft-contrib/pull/17) would allow, so
  this RFC stays the record of the decision? This RFC keeps them inline until
  RFC-0013 is accepted.
- Should `string-error` warn rather than fail for an app that still has
  existing hits, so it can be turned on before the cleanup is finished?
- Should `--json` be required on every subcommand that prints a listing, or
  only where a program is expected to read it?

## Appendix A: the CLIs on 2026-11-26

Source read at each repository's `main`; behaviour from the latest non-rc Linux x86_64 release's
`bin/<app>-cli`, run with an empty HOME and no input. "Sub flag" is an unknown flag given to a
subcommand (`info` for the clap CLIs and pdfcraft, `commands` for the others). Exit codes are for an
unknown top-level flag.

| App | Crate | Parser | thiserror | `run(args, out, err)` | `-V` | `version` word | Exit | Sub flag | Linux completions and man |
|---|---|---|---|---|---|---|---|---|---|
| anicraft | `crates/cli` | hand | no | no | yes | accepted | 1 | exit 1 | no |
| cadcraft | `apps/cadcraft-cli` | clap | yes | yes | yes | refused | 2 | refused, 2 | yes |
| chatcraft | `apps/chatcraft-cli` | hand | no | yes (`cli::main`) | n/a (no release) | n/a | n/a | n/a | n/a |
| craft-anything | `apps/craft-anything-cli` | clap | no | no | yes | refused | 2 | refused, 2 | no |
| deckcraft | `apps/deckcraft-cli` | hand | no | no | yes | accepted | 1 | ignored | no |
| designcraft | `apps/designcraft-cli` | hand | no | no | yes | accepted | 1 | ignored | no |
| effectcraft | `apps/effectcraft-cli` | hand | no | no | yes | accepted | 2 | refused, 2 | no |
| figcraft | `apps/figcraft-cli` | clap | yes | no | yes | refused | 2 | refused, 2 | no |
| filmcraft | `apps/filmcraft-cli` | hand | no | no | yes | accepted | 2 | ignored | no |
| gridcraft | `apps/gridcraft-cli` | hand | no | no | yes | accepted | 1 | refused, 1 | no |
| lightcraft | `apps/lightcraft-cli` | hand | no | no | yes | accepted | 1 | ignored | no |
| loopcraft | `apps/loopcraft-cli` | hand | no | no | n/a (no release) | n/a | n/a | n/a | n/a |
| materialcraft | `apps/materialcraft-cli` | hand | no | no | yes | refused | 1 | refused, 1 | no |
| pdfcraft | `apps/pdfcraft-cli` | hand | no | no | **no** | refused | 1 | refused, 1 | no |
| photocraft | `apps/photocraft-cli` | clap | yes | yes | yes | refused | 2 | refused, 2 | yes |
| soundcraft | `apps/soundcraft-cli` | clap | yes | yes | yes | refused | 2 | refused, 2 | yes |
| texturecraft | `apps/texturecraft-cli` | hand | no | no | yes | accepted | 2 | refused, 2 | no |
| vectorcraft | `apps/vectorcraft-cli` | hand | no | no | yes | refused | 1 | ignored | no |
| wordcraft | `apps/wordcraft-cli` | hand | no | no | yes | refused | 1 | refused, 1 | no |

Other differences the adoption pull requests meet:

- anicraft's overwrite flag is `--overwrite yes` (on `gif`), not `--in-place`.
- craft-anything exits 130 on Ctrl-C. WordCraft exits 3 for a strict-mode report.
- With no arguments, 5 print help and exit 0, 4 exit 1 and 8 exit 2.
- TextureCraft prints `TextureCraft 0.6.1 (…)` for `--version`, without `-cli`.
- figcraft's CLI depends on `clap_complete` and `clap_mangen`, but its release ships no completions or
  man pages.
