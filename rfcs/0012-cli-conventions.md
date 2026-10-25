# RFC-0012: Command-line conventions

## Summary

Every Craft app's command-line tool (`<app>-cli`) behaves the same way and is
built the same way: [clap](https://docs.rs/clap) derive types that parse the
command line into typed values, [thiserror](https://docs.rs/thiserror) errors,
the same exit codes and output streams, and generated shell completions and man
pages. A CLI that follows these rules says so ("follows RFC-0012"), so a user
who knows one Craft CLI knows them all, and a reviewer can check a CLI against
one list instead of re-arguing each choice per app.

The rules come from photocraft's move to clap
([photocraft#7071](https://github.com/storytold/photocraft/issues/7071), merged
in [#9167](https://github.com/storytold/photocraft/pull/9167) and
[#9194](https://github.com/storytold/photocraft/pull/9194)), which is the
reference implementation.

## Motivation

Five other Craft apps ship a CLI (cadcraft, deckcraft, filmcraft, gridcraft
and wordcraft), and none of them uses clap: each parses its arguments its own
way. The problems photocraft had before its port are likely in each of them:

- A mistyped flag (`--fromat`) was silently ignored, and a repeated one quietly
  took the last value.
- A bad value failed late, after work had started, or not at all.
- Errors were strings, so a caller couldn't tell a usage error from a failure,
  and messages drifted from flag to flag.
- `--help` was hand-written text that drifted from what the parser accepted.

Fixing these per app, with different answers each time, would give users five
different CLIs and reviewers five different sets of rules to remember.
Photocraft has already answered the questions, through review by several
maintainers, so the other apps can follow it.

## Proposal

### 1. What a Craft CLI looks like

1. **Name and shape.** The binary is `<app>-cli`, with subcommands
   (`<app>-cli convert …`). `--help`/`-h` and `--version`/`-V` work everywhere;
   `help <subcommand>` works too. There is no `version` subcommand.
2. **Exit codes.** `0` success, `1` the command failed, `2` usage error.
3. **Streams.** Results go to stdout: JSON when a program is meant to read them
   (pretty-printed by default, one line with `--compact` or as the command
   documents). Errors, warnings and progress go to stderr, errors as
   `error: <message>`. A subcommand that speaks a protocol on stdio (an MCP or
   JSON-lines server) writes nothing else to stdout.
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
     authority comes from flags only.
8. **Help is the documentation.** The help text is the doc comments on the clap
   types, so `--help` is the source of truth. The app's book has one page for
   its CLI that links to it rather than repeating it.
9. **Shell completions and man pages** (bash, zsh, fish, elvish, PowerShell, and
   a page per subcommand) are generated from the same clap command by
   [clap_complete](https://docs.rs/clap_complete) and
   [clap_mangen](https://docs.rs/clap_mangen) in an example or build step, and
   the Linux and FreeBSD packages install them. The generators are dev or build
   dependencies, not part of the binary.

### 2. How it is built

1. **Data first: invariants live in types.** The command line is parsed into
   clap derive structs whose fields already have their final types: enums for
   a fixed set of values, ranged integers, `PathBuf`, `SocketAddr`, parsed
   JSON. Relations between flags are declared with clap (`ArgGroup`,
   `conflicts_with`, `requires`). The code that does the work never sees a raw
   string it has to validate.
2. **Errors are enums.** Each error a crate returns is a variant of a
   `thiserror` enum. No `Result<_, String>`, no `Err(format!(…))`, no message
   kept in a `&str` constant. A usage error the parser can't express is still a
   variant, turned into a clap error by a method that picks its
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
7. **Tests pin the interface**: help on stdout with exit 0, each usage error's
   exit code and stream, and the result of each subcommand.

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
"Follows [RFC-0012](https://github.com/storytold/craft-contrib/blob/main/rfcs/0012-cli-conventions.md)"
to the CLI's book page, listing any rule it doesn't follow yet and why.

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
