# Testing

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as in RFC 2119 and RFC 8174.

These rules apply to every test in the Craft repositories, written by a person or an agent. `AGENTS.md` carries the short form and links here. Changing a rule needs an RFC.

## Which sections to read

- **Writing a new test:** What tests are for, The keep rule, Cut categories (a new test SHOULD NOT fall in one), Fix categories, Where new tests go, Hostile input, Fixtures, Tests that can't run here, Flaky and hanging tests.
- **Judging or cutting existing tests:** How to judge a test, Cut categories, Fix categories, Examples of kept tests, Cutting existing tests. Existing tests SHOULD be judged and cut only when an issue asks for it.

Three rules hold in both cases:

- You MUST NOT delete, weaken, ignore or skip a test to make a check pass.
- Tests MUST be cut or quarantined only in a pull request of their own that lists every removed test.
- You MUST NOT quarantine a test that fails every time.

## What tests are for

Tests specify the invariants of the code: what must stay true however the code changes. Each test SHOULD state one invariant, and fail when a plausible change breaks it.

Every branch of the code SHOULD be reached by an invariant test or be simple enough to check by reading. For a branch that no test reaches, decide which of three it is:
- **A missing invariant:** often an error or edge path in a parser or format reader. Write the test.
- **Code simple enough to check by reading:** leave it.
- **Dead code:** delete it.

Coverage points at gaps. It is not a target to reach with tests written for their own sake.

## The keep rule

A test SHOULD stay only if you can write all three of these:

1. **Invariant:** what it guards, in one line.
2. **Breaking change:** a plausible change to the code that breaks that invariant. Plausible means a change someone could make while fixing, refactoring or extending this code: reordering steps, an off-by-one, a dropped case, a changed default, a replaced algorithm. Deleting the function or arbitrary sabotage doesn't count.
3. **Only this test:** no other test, and no shared-suite case, would fail on that change. Find the other tests (grep for the function) and check.

If you can't write all three, or are unsure whether you can, the test SHOULD go: the burden is on keeping a test, not on cutting it. Apply this before you add a test. Existing tests are judged only as described under "Cutting existing tests".

The keep rule is necessary, not sufficient: a check in a cut category SHOULD go even when you can write all three lines. For example, readable code (C1) has plausible breaking changes too, but a reviewer catches them by reading.

The cut categories are the common reasons a test fails the keep rule. Name the one that applies. If none fits but the keep rule still fails, write `CUT K` and say which of the three lines you couldn't write.

## How to judge a test

Judge from the test's body and the code it drives, never from its name or file alone.

1. **Split it into checks.** A check is an assertion, or a group of assertions about one thing. Most tests have one check; some have several.
2. **Judge each check:** does a cut category apply, and does it pass the keep rule?
3. **Give a verdict for the test:**
   - Every check is cut: `CUT` with the categories.
   - Some checks are cut, at least one stays: `TRIM`. Delete the cut checks; the test keeps the rest. Rename it if its name no longer fits.
   - No check is cut, but a fix category applies: `FIX`. Rewrite it so that none applies, then judge it again.
   - Otherwise: `KEEP`, with the invariant and the breaking change.

Tests behind a feature flag or a target OS are judged like every other test. A test marked `#[ignore]` that no CI job runs is C8: in practice it checks nothing. A flaky test quarantined under an open issue (see "Flaky and hanging tests") is judged when that issue closes.

**Cite only a test that stays.** A covering test (for C6, or for the "only this test" line) SHOULD itself be kept or trimmed, with the covering check still in it. Two tests that cover each other can't both go: keep the one with more reach, and keep the shared check in it.

Write the verdict as one line per test (the paths, lines and names below show the format only):

```
crates/draw/tests/it/crop.rs:41  swap_exchanges_width_and_height  CUT C1
crates/ui-widgets/tests/it/gradient.rs:88  keys_move_delete_and_add  CUT C2
crates/xlsx/src/xlsb/ftab.rs:500  function_numbers  TRIM C3 -> keep the sorted-table check, drop the entry spot checks
crates/ui-contract/src/style/contrast.rs:210  wcag_contrast  KEEP guards: the WCAG 2 formula every contrast check relies on; breaks if: the luminance weights or the 0.05 offsets change
crates/ui-egui/src/frame_stages.rs:336  two_30_ms_passes  FIX F2 -> assert the first pass's input time survives the second pass
```

## Cut categories

A check in a cut category SHOULD NOT be written, and SHOULD be cut from an existing test when a cutting pull request is asked for.

### C1 Readable
**Question:** Can a reviewer confirm the code under test is right by reading it, because it is a few lines with at most one branch and no state carried between calls?
- "The code under test" is the code the check exercises, not the whole function around it: a limit check inside a long function is judged by the comparison and its effect.
- **Example:** `swap_exchanges_width_and_height`; the function is two assignments.
- **Also C1:**
  - a bug fix whose fixed code is a few lines with at most one branch and no state between calls;
  - a limit enforced by one comparison or one `take(n)`.
- **Not this:**
  - code with a loop that carries state, several interacting branches, or arithmetic whose result isn't obvious, such as geometry, rounding or layout sharing;
  - a limit that interacts with other state, such as one allowance shared across several parts.

### C2 Wiring
**Question:** Does the check only show that an input reaches a function: a key calls a model function, a button runs its command, a field or hook is passed through, a menu item maps to a command id?
- **Example:** `keys_move_delete_and_add_stops_one_set_each`; the key table *is* the mapping.
- **Also C2:** a shared mechanism called from several entry points. Test the mechanism once, where it is defined. Each entry point's call to it is wiring.
  - **Example:** a modal gate checked through the menu bar's click, its keys, the native menu's clicks and the Alt key: one test of the gate stays; the four entry-point checks go.
- **Not this:** routing logic that decides *which* receiver gets the input (focus, key routing, priority). That logic has rules, and a rule can break.

### C3 Default, constant or table entry
**Question:** Does the check assert a default value, a constant, a getter or setter, or an entry of a table or `match`, with no logic between the value and the assert?
- **Example:** a test that a new document's resolution is 72 ppi, where `new()` just sets that field.
- **Example:** spot checks that `Key::A` maps to `Code::KeyA` in a key table.
- **Not this:**
  - a default computed from other inputs, such as a size that follows the system's DPI;
  - a property of the whole table that other code relies on: sorted for a binary search, an entry for every variant, no duplicates.
- A table entry stays C3 even when a bug fix added it, and even when its value comes from outside our code (a spec, another program, a platform). A wrong entry is caught by checking the table against its source, not by one spot-check test per entry.
- A default limit is C3 unless the code computes it. A comment explaining where the number comes from is not computation.

### C4 Compiler-checked
**Question:** Would the code fail to compile if the asserted property were false? Examples: a builder returns the right type, an enum has a variant, a type is `Send`.
- **Not this:** a runtime property the types don't express.

### C5 Library behaviour
**Question:** Is the behaviour that decides pass or fail implemented in a dependency (egui, std, serde, ICU, unicode-segmentation, any crate outside this repository), with our code only calling it?
- If our code around the call is itself C1-readable (a lookup, an offset added, a feature switch), the check is C5.
- **Example:** a test that `serde_json` round-trips a plain derived struct.
- **Example:** a test that a mirroring function returns `)` for `(`, where the function is one call into the Unicode crate.
- **Not this:** our own adapter logic around the library that isn't C1: a custom `Deserialize`, a migration, error mapping with branches, offset mapping across several runs.
- A crate in this repository is not a library. Its behaviour is tested in that crate; a test of it elsewhere is C6 or belongs there.

### C6 Duplicate
**Question:** Does another test, or a shared-suite case, already go through the same code and fail on the same breaking change? Find that test and confirm it before citing it.
- Of two overlapping tests, keep the one that reaches more of the code. At equal reach, keep the one at the level the user meets the behaviour (through the component), and cut the other.
- **Example:** `a_moved_stop_passes_its_neighbours_and_keeps_its_tag` (stop model) is covered by the editor's drag test through the same move function.
- **Example:** `a_window_blur_cancels_a_captured_press_once` is covered by the shared suite's press-cancel case on every component.
- **Not this:** a test that reaches a branch the other test doesn't. If only part of it overlaps, TRIM the overlapping checks.

### C7 Combination
**Question:** Is the check one of several that differ only in input values and go through the same branches?
- Keep one per distinct branch; cut the others.
- **Example:** five tests of a unit converter, one per unit, all through the same multiply.
- **Not this:** values that land in different branches or edge cases: zero, negative, overflow, empty, the boundary of a clamp.
- Where both C6 and C7 fit, cite either.

### C8 No real check
**Question:** Does the check run code without asserting anything a regression would change? For example, it builds and renders a component and only checks that it didn't panic.
- **Not this:** "never panics on hostile input" for a parser or decoder. There, not panicking *is* the invariant.

## Fix categories

A test in a fix category SHOULD be rewritten so that none applies.

### F1 Exact value
**Question:** Does the check assert exact pixels, colours, strings, float values or bytes, where the exact value is our own design choice?
- **Fix:** assert the relation that matters, such as "inside", "wider than", "changed", "same as before".
- **Not this:** a value fixed outside our code, which is a specification:
  - bytes a file format requires, and a standard's reference values (the WCAG contrast examples);
  - the output of another program we must match (the `.xlsx` text Excel writes for a value a reader decodes);
  - a platform convention (Windows names it "Exit", macOS "Quit");
  - bundled data (a hyphenation dictionary's entries);
  - golden files of a writer whose output other programs read.

### F2 Timing
**Question:** Does the result depend on wall-clock time, `sleep`, or how fast the machine is?
- **Fix:** drive the clock and frames by hand (a manual clock, `settle()`), and assert what happened, not how long it took. Give randomized tests a fixed seed. To stop a test that might hang, run it under craft-testkit's `with_deadline`; the deadline ends the test and is not asserted.

### F3 Platform-fragile
**Question:** Could the test pass on one operating system and fail on another, without the code under test having platform-specific behaviour?
- **Typical causes:**
  - absolute path literals such as `/m/a.png`;
  - byte comparisons of checked-in text files, which Windows checks out with CRLF;
  - UI tests of PC-only behaviour that set no OS;
  - fixtures sized for one screen;
  - floats compared exactly across CPUs.
- **Fix:**
  - build paths with `Path::join` or `cfg!(windows)` constants;
  - add `text eol=lf` to `.gitattributes`;
  - set the OS with `fixture.os(...)`;
  - fit the content to the room the fixture gives;
  - compare floats with a tolerance or by relation.

### F4 Unbudgeted stress
**Question:** Does the test loop over many generated or random cases, or build inputs of millions of items, without going through the shared case budget?
- **Fix:** use the shared caps (`PROPTEST_CASES`, and `CRAFT_HOSTILE_DIVISOR` through craft-testkit's `hostile::hammer`), or test a limit with a small limit value instead of a huge input. Pull requests then run a fraction of the cases, and the nightly run does the full count.

### F5 Slow
**Question:** Does a unit test take over 0.1 s, or an integration test over 1 s, in the debug build CI runs, without a comment saying why?
- Take the time from the test run (nextest prints it), not from reading the code.
- **Fix:** make it smaller or faster, or put it under the stress budget (F4).
- If the time goes into shared setup (decoding bundled data, building a large fixture), the fix is in the setup: build it once and share it between tests.

### F6 Not a code test
**Question:** Does the test check the repository rather than run code: manifest text, a file listing, a licence field?
- **Fix:** move it into the lint scripts.

## Where new tests go

- One integration test binary per crate SHOULD be used: `tests/it/main.rs`, with the test files as its modules (`tests/it/<area>.rs`). Every `.rs` file directly in `tests/`, and every `tests/<dir>/main.rs`, is a separate binary that links the crate and its whole dependency tree again, on every operating system CI runs. A test that needs a feature goes behind `#[cfg(feature = "...")]` inside that binary, not in a binary of its own.
- Test a shared mechanism once, where it is defined, not in every caller.
- Test behaviour through the component the user meets, not through the helper it calls.
- A component SHOULD get its own test only for behaviour unique to it. Everything every component does is the shared suite's job: focus, disabled, right-to-left, press and cancel, themes, text size.
- One invariant per test, named as a sentence about it: `a_stop_keeps_its_place_among_stops_at_its_location_unless_it_crossed_them`, not `test_stops_3`.
- A bug fix SHOULD come with a test that fails without the fix, unless the fixed code is a few lines with at most one branch and no state between calls.
- A crash fix SHOULD add the input that panicked to the crate's hostile-input sweep, also when the fixed code is that small. A crate with no sweep SHOULD get a test that panicked before the fix.
- A storage or format migration SHOULD get a test that each value the old version wrote survives the switch.
- A failing generated case (proptest, fuzzer) SHOULD become a named regression test. It SHOULD NOT be committed as a regression file.

## Hostile input

Parsers, decoders and command handlers SHOULD return an error for any input, and SHOULD NOT panic or hang on it: damaged files, junk command parameters, malformed protocol messages. "Never panics" is the invariant here (C8 does not apply).

- Each app SHOULD keep one sweep that runs every command with hostile parameters, and each parser or decoder SHOULD have a mutation sweep (truncation, bit flips, corrupt sizes). A crate that depends on craft-testkit SHOULD run its mutation sweep through `hostile::hammer`.
- A new command, parser or protocol method SHOULD be added to the sweep in the same change.
- Sweeps SHOULD go through the shared case budget (F4).

## Fixtures

- Fixtures SHOULD be committed and use real-format data; a mock only where no real-format sample exists. A crash regression SHOULD use a synthetic file cut down to the bytes that trigger it.
- Every fixture follows the fixture rules of [clean-room.md](clean-room.md), provenance included.
- Real-file corpora live in a corpus repository (for example `storytold/photocraft-corpus`), fetched pinned by commit and checked by sha256. A binary fixture SHOULD be under 100 KB. A larger one SHOULD go in a corpus repository and SHOULD NOT be committed to an app or library repository.

## Tests that can't run here

- A test that needs something the machine lacks (a GPU, a corpus file, a craft-fonts face, an oracle program) SHOULD skip with a reason the run lists. It SHOULD NOT skip with a bare `return` or `eprintln!`.
- A `*_REQUIRED` variable turns that skip into a failure: unset or empty means skip, `1` means required, and any other value fails with a message naming the variable and the accepted value. CI sets it where the resource is provided.

## Flaky and hanging tests

- A test that fails and then passes on a rerun of the same commit is flaky. Open an issue with the failing log. You SHOULD NOT rerun until it passes.
- If it failed on two unrelated pull requests, it MAY be quarantined with `#[ignore = "flaky: #<issue>"]` in a pull request of its own, until the issue is fixed.
- A test that fails every time MUST NOT be quarantined: fix the code or the test.
- A test that hangs is a failure.

## Examples of kept tests

- **`a_stop_keeps_its_place_among_stops_at_its_location_unless_it_crossed_them`.** Guards: tie order at equal locations depends on the direction of travel. Breaks if: the sort is changed to an unstable one or drops the direction. No other test checks ties.
- **`wcag_contrast`.** Guards: the formula every contrast check relies on. Breaks if: a luminance weight changes. Every other contrast check uses the same function, so a wrong formula would pass them all.
- **`disabled_cells_take_no_input`** (crop bar fields). Guards: disabled reaches every cell. Breaks if: a new cell is added without the disabled flag. The fields are a pattern, not a component, so the suite's disabled case never runs on them.
- **`zoo_decodes_and_never_panics_when_cut_or_corrupted`** (TIFF). Guards: hostile input never panics a decoder. Breaks if: a length read loses its bounds check. It is F4 if it runs its full corpus on every pull request.

## Cutting existing tests

- A cut SHOULD be done per crate, one pull request each, containing no other change.
- The pull request MUST list every removed or trimmed test, with its category, one line each, in the format above. A maintainer can then object to a single line instead of the whole cut.
- Judging a whole file SHOULD start from the code rather than the tests: list what in the code could plausibly break unnoticed, then map each test to one of those items or to a category.
