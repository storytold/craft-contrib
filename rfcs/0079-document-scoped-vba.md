# RFC-0079: Document-scoped VBA, starting with user-defined functions

Related: storytold/gridcraft#6 and #1143 (VBA requests), storytold/gridcraft#990,
#143 and #3332–#3334 (VBA pull requests), storytold/wordcraft#172, #2586 and
#5483; RFC-0015 (confined file access), RFC-0040 (shared library first),
RFC-0046 (shared plug-in runtime, open).

Affects: GridCraft and WordCraft, and one new shared crate in craft-libs.
DeckCraft can adopt the same crate later; this RFC does not ask it to.

## Summary

Today VBA is out of scope in GridCraft "by owner decision", and WordCraft lists
"no VBA execution (macros preserved only)" as an open gap. Both apps keep a
file's VBA project on save and never run it.

This RFC proposes a narrower scope than "support VBA": **document-scoped VBA**.
That means VBA code that reads and changes only the document it lives in. Code
that drives the operating system, other applications or arbitrary files stays
out of scope.

The first step is **user-defined functions (UDFs) in GridCraft**: VBA
`Function` procedures called from worksheet formulas. Excel already forbids a
UDF called from a cell to change anything except its own result, so this step
needs no restriction that Excel itself does not already impose.

Later steps, each needing its own approval on this RFC's terms, are
document-scoped `Sub` macros in GridCraft and WordCraft.

## Motivation

### Documents travel between suites

A Craft app is useful to its users only if the documents they receive from
other people open and work, and if the documents they send back keep working
for people who use Microsoft Office. The suite already invests heavily in this:
round-trip fidelity checks, keeping a `.xlsm` or `.docm` VBA project intact on
save (storytold/gridcraft#4468 and #4522 are recent fixes), compatibility
behaviour for legacy formulas.

VBA is the one part of that exchange that is preserved but does not work. A
recipient of a workbook can see and edit it in GridCraft, but any cell that
calls a VBA function cannot be recalculated. At best the cell keeps the value
Excel last saved (as storytold/gridcraft#2192 proposes for XLS), and that value
silently goes stale when an input changes. In a financial model that is worse
than an error.

### Two kinds of VBA, and only one of them matters here

VBA in the wild falls into two groups:

1. **Workplace-bound applications.** Code that opens files on a network share,
   consolidates thirty branch workbooks, sends mail through Outlook, calls a
   DLL or shells out to a script. These are programs that happen to use Excel
   or Word as their runtime. They depend on one company's machines, paths and
   installed software. Nobody emails them to a counterpart. If someone did,
   they would not work on the recipient's computer even in Office. A company
   moves them only when it moves its whole platform. They are not a document
   interoperability problem, and this RFC leaves them out.

2. **Code that belongs to the document.** Functions and macros that compute,
   validate, format, navigate, protect or reset the document they live in.
   They are written because the host lacked a built-in function, or because the
   formula-only version would be fragile, slow or unreadable. These are sent to
   clients, counterparts, auditors and regulators as part of the document. This
   is where compatibility is decided.

The second group is common in finance, insurance, engineering and research.
Typical examples are custom interpolation, curve bootstrapping, day-count
conventions, option pricing and special amortization schedules. In these
fields a spreadsheet that cannot run its UDFs is not a working copy of the
model.

The existing reasons for keeping VBA out of scope are sound for the first group:
the size of the full object model, the security record of macros, and the long
tail of COM and Windows APIs. They do not apply to the second group in the same
way.

### Office itself now blocks macros in shared files

Since 2022, Office blocks VBA macros by default in files that come from the
internet, including email attachments
([Microsoft documentation](https://learn.microsoft.com/deployoffice/security/internet-macros-blocked)).
A recipient has to unblock the file outside Office before its macros run.
Shared documents with VBA therefore already have a rough path in Office. A
Craft app that can run document-scoped code under a clear, narrow sandbox, and
that can tell the user what that code will touch, can offer a better experience
for the same file, not a worse one.

### The pieces partly exist

- MS-OVBA decompression and compound-file reading are proposed as a read-only
  `gridcraft-vba` crate (storytold/gridcraft#3332–#3334).
- WordCraft has a proposal for a read-only view of macro modules
  (storytold/wordcraft#5483).
- storytold/gridcraft#990 implements a VBA interpreter, object model and
  UserForms in one ~13,400-line pull request. It has had no review.

The volume of independent attempts shows demand. Their state shows that without
an agreed scope none of them can be reviewed or merged. This RFC is meant to
provide that scope.

## Proposal

### 1. Scope: document-scoped VBA

Code is **document-scoped** when, as run by a Craft app, it can only:

- read and change the document it belongs to, through the host's own commands;
- read its arguments and return values;
- interact with the user through the host's dialogs;
- read or write a file **only** through a file dialog the user answers, or
  inside the roots granted under RFC-0015.

Everything else is **out of scope** and fails with a clear message naming the
unsupported statement:

- `Shell`, `Declare` (DLL calls), `CreateObject` and `GetObject` (COM);
- network access;
- opening, saving or deleting files by path outside the rules above;
- `Application.OnTime` and other timers;
- the VBA project object model (code that reads or edits code);
- add-in files (`.xlam`, `.dotm` templates loaded as add-ins).

### 2. Phase 1: user-defined functions in GridCraft

- **Language:** a subset of the VBA language as specified in
  [MS-VBAL](https://learn.microsoft.com/openspecs/microsoft_general_purpose_programming_languages/ms-vbal/d5418146-0bd2-45eb-9c7a-fd9502722c74).
  It covers value types, arrays, control flow, procedures in the same project,
  `Select Case`, `On Error`, and `CVErr` for worksheet errors. MS-VBAL is a
  public specification written to enable independent implementations, so this
  follows standards/clean-room.md.
- **Object model:** only what a UDF needs:
  - `Range` arguments, read-only: values, size, cells, rows and columns;
  - `Application.WorksheetFunction`, mapped to GridCraft's own functions;
  - `Application.Volatile`;
  - `Application.Caller`, read-only;
  - returning an array, which spills like any dynamic array.

  Other object members a UDF calls return `#VALUE!` with a message.
- **Excel's own rule applies:** a UDF called from a cell cannot change the
  workbook, as
  [Excel documents](https://support.microsoft.com/kb/170787).
  The interpreter enforces this, so phase 1 has no side effects by
  construction.
- **Limits:** UDFs run in-process under fuel (step) and memory limits, like the
  plug-in runtimes described in RFC-0046. A UDF that exceeds them returns an
  error. It never hangs recalculation.
- **Where the code lives:** the reader (MS-OVBA, MS-CFB) and the interpreter go
  into craft-libs under RFC-0040, because GridCraft, WordCraft and later
  DeckCraft read the same VBA storage. The worksheet binding (`Range`,
  `WorksheetFunction`, the calc engine) stays in GridCraft.
- **Saving:** the VBA project is still written back unchanged. Running a UDF
  never changes the project.

### 3. Phase 2: document-scoped macros in GridCraft and WordCraft

Phase 2 needs its own approval as an update to this RFC. That approval should
cover how phase 2 meets these requirements:

- **One run, one undo step.** The host snapshots the document, runs the macro
  and records one undo step. On an unhandled error it restores the snapshot.
  GridCraft already does this for its own typed macros (`docs/macros.md`).
- **Preview.** Before running, the host can run the macro on a copy and show
  what it would change. The user sees "this macro edits sheets A and B" rather
  than deciding whether to trust the whole file.
- **Runtime state.** Module-level and `Static` variables are part of the
  snapshot, or are reset on undo, so that undo leaves the code and the document
  consistent.
- **Events** (`Workbook_Open`, `Worksheet_Change`, `Document_Open`) are off by
  default. An event that runs is part of the transaction of the action that
  caused it.
- **Files** only as defined in section 1. A file written by a macro is staged and
  committed only when the run succeeds, so a failed run leaves no partial file.
- **User interface:** `MsgBox` and `InputBox` first. UserForms later, as a
  separate decision.

### 4. Trust and naming

- A document's VBA never runs until the user allows it for that document,
  through each app's existing trust mechanism. Whether phase 1 UDFs, which
  cannot change anything, need the same prompt is an open question below.
- Feature labels follow standards/naming.md. "VBA" may appear where it names
  the language and file content we support (a compatibility statement). The
  feature's own labels use the suite's words, for example "Macros" and
  "Custom functions".

### 5. What this RFC decides

Merging this RFC means:

- document-scoped VBA, as defined in section 1, is **in scope** for GridCraft
  and WordCraft;
- phase 1 pull requests can be reviewed against this text;
- the "out of scope" lines in GridCraft's `docs/target-app-parity.md` and
  `docs/gaps.md`, and WordCraft's `docs/gaps.md`, are updated to say
  "workplace-bound VBA is out of scope; document-scoped VBA follows RFC-0079".

It does not decide phase 2, and it does not commit any maintainer to writing the
code.

## Conformance

A workbook saved by Excel stores the last calculated value of every cell. For a
cell that calls a UDF, that value is a reference result for the inputs in the
file. Phase 1 tests:

- recalculate every UDF cell in corpus files and compare with the stored value;
- cover the places where VBA arithmetic differs from worksheet arithmetic:
  - `Round` (round half to even) versus `ROUND`;
  - `Mod` (operands rounded, sign of the dividend) versus `MOD`;
  - `Currency`, `Integer` and `Long` overflow, integer division `\`;
  - date serials before 1 March 1900.

Corpus files follow the fixture rules in standards/clean-room.md.

## Alternatives

- **Keep VBA out of scope.** Shared models with UDFs stay broken in Craft
  apps, and new VBA pull requests keep arriving with no scope to review them
  against.
- **Full VBA** (the approach of storytold/gridcraft#990). It can't be reviewed,
  it is not portable to the web build, and it reopens every security question
  phase 1 avoids.
- **Translate VBA into formulas or typed macros.** An agent could rewrite a UDF
  as a `LAMBDA`, or a macro as a GridCraft typed macro. This is a useful
  optional tool, but it is not a replacement:
  - not every UDF can be expressed as a formula. Examples are code that reads
    cell formatting, keeps `Static` state, or relies on VBA's own rounding;
  - equivalence can only be tested, not proven;
  - translation changes the file the sender gave us.
- **Run VBA through the WebAssembly plug-in runtime (RFC-0046).** VBA would
  need an interpreter compiled to WebAssembly, run inside another interpreter.
  The proposal shares that runtime's limits and permission vocabulary instead of
  its execution engine.

## Open questions

1. Should phase 1 UDFs, which cannot change the document, run without the
   document trust prompt, or behind the same prompt as macros?
2. Should UDFs be allowed to read formatting of their arguments (number format,
   fill colour)? Excel allows it, and some common UDFs depend on it, but it ties
   recalculation to formatting changes.
3. Who maintains the interpreter in craft-libs? Phase 1 is only worth merging
   with a named owner.
4. Should DeckCraft be named as an adopter now, or left to a later RFC?
