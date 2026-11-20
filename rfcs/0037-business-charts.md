# RFC-0037: Business charts in DeckCraft and GridCraft

## Summary

DeckCraft and GridCraft gain the business charts that PowerPoint and Excel users usually get from
add-ins:
- stacked, clustered and 100% charts with total and difference decorations;
- bridge-style waterfalls with stacked segments and computed totals;
- Mekko charts;
- scatter charts with quadrant lines and captions;
- rounding that keeps shown parts adding up to shown totals;
- charts in a deck that stay linked to a range in a workbook.

One new craft-libs crate, `craft-bizchart`, turns a chart specification into a display list: rectangles,
paths, text runs and connectors, in points. Each app draws that list with its own renderer and keeps it
in its own document model. DeckCraft and GridCraft are the two consumers RFC-0009 asks for.

Every action is a command, so the CLI, the control port and MCP get all of it. A `deckcraft-cli fill`
subcommand builds decks from a template plus JSON data.

Beyond the crate, this RFC fixes the few things the two apps must agree on: the chart spec, the link
record a deck keeps to a workbook range, and the rules for CAGR, rounding and number formats.

## Status: a working prototype

The first milestone runs on my machine and isn't public yet:

- **The crate** (`ChartSpec` v1 in JSON):
  - stacked, 100% and clustered charts, as columns and as bars;
  - segment and total labels, multi-line labels, and label colours picked for contrast with their fill;
  - number formats carried in the spec;
  - rounding that preserves totals.

  Every other kind returns `LayoutError::UnsupportedKind`. It is tested by geometry and by property tests,
  never by pixels.
- **DeckCraft:**
  - a business-chart element: a group of native shapes with its spec in the shape's `p:custDataLst` tag,
    plus an `.xlsx` copy of its data inside the `.pptx`;
  - its commands, and an Insert menu entry;
  - links to workbook ranges with a Links pane, update on open, and `deckcraft-cli run --recalc`.
- **GridCraft:**
  - a business chart from a selected range, drawn in the sheet;
  - saved in `.xlsx` as a native chart Excel can display, plus the spec. Excel opens the file cleanly and
    it round-trips;
  - an Insert CAGR command;
  - worksheet functions for rounding that preserves totals.

I can open the crate as a draft PR against craft-libs during the comment period, so reviewers can read the
code before deciding.

## What the apps have today

**DeckCraft:**
- It reads, draws and saves native charts and the `chartex` types (`docs/file-format-parity.md`).
- Editing chart data is a P0 row that is still partial (`docs/gaps.md`).
- It keeps `p:custDataLst` tags through a round trip (storytold/deckcraft#531, `tags_survive_round_trip`).
- Its roadmap lists add-ins as out of scope (`ROADMAP.md`).

**GridCraft:**
- It draws 26 of Excel's roughly 60 chart subtypes, including waterfall and funnel
  (`docs/chart-parity.md`).
- A bar or column value axis always starts at 0 (storytold/gridcraft#1688), and data labels are one on/off flag.

**Neither app has:**
- a Mekko chart, a quadrant scatter or a waterfall with stacked segments;
- rounding that keeps parts adding up to totals;
- a way to keep a deck chart's data tied to a sheet range.

People who use these charts today get them from add-ins, and DeckCraft doesn't plan an add-in model. So
the capability has to be native.

## Why a shared crate

Both apps need the same layout maths: bar geometry for each kind, label placement and collision handling,
decoration geometry, and rounding. A chart made in GridCraft from a range and placed in DeckCraft has to
mean the same thing in both. Two private copies would drift, which is the problem RFC-0030 describes.

The crate is pure data and algorithms, with no UI, no I/O and no app types. That is the shape RFC-0009
admits (§3, §4).

## Architectural boundaries

**The crate.** `craft-bizchart` depends on `serde` and `serde_json`, and on nothing from any app, `egui`
or `craft-ui`.
- **Inputs:**
  - **`ChartSpec`**: the kind, the data table, decorations, a style reference with any hand-set colours,
    label options with number formats, and an optional link record. It is versioned JSON. New fields are
    optional, so `v` stays 1; a `v: 2` spec is refused with a clear error, and unknown keys are kept.
  - **A text measurer and a number formatter, supplied by the app.** Labels use the app's fonts, and the
    app renders each `NumberFormat` (scale, prefix and suffix, sign, decimals, separators). The crate
    ships a small reference formatter for tests and simple hosts.
- **Output:** a **display list** whose path segments are lines and quadratic and cubic Béziers, as in
  kurbo, plus the plot area's frame. Moving onto RFC-0030's `PathData` later is mechanical, and this RFC
  doesn't depend on RFC-0030 landing.
- **Limits:** it builds for wasm32, follows the house-rules `deny` list, bounds every allocation by the
  spec's size, and never panics.

**DeckCraft.**
- A business chart is a **group of native shapes plus its spec in a `p:custDataLst` tag**. storytold/deckcraft#2135 proposes
  the same carrier for Harvey balls and checkboxes, so DeckCraft has one way to store generated elements.
- In DeckCraft the group is live: changing data or decorations regenerates the shapes.
- In other presentation software it opens as ordinary grouped shapes that look right but aren't editable
  as a chart. DeckCraft's file-format docs record how it is written, as house rule 16 asks.
- The data is also saved as an `.xlsx` part inside the `.pptx`, so any spreadsheet can open it.
- The native `c:chart` model is left alone. It can't express Mekko or the decorations, and this RFC
  doesn't replace it.

**GridCraft.**
- `businessChart.fromRange` builds a spec from a selection, with a link record to that range.
- The chart floats in the sheet, drawn from the shared display list.
- In `.xlsx` it is saved as a native chart that other spreadsheet apps display, plus its spec in a hidden
  defined name, so GridCraft can bring the business chart back.

## What the two apps agree on

These are the cross-app decisions. Each has a test in the crate, and in each app that implements it.

1. **The spec.** `ChartSpec` and its JSON live in the crate. Both apps read and write it unchanged.
2. **The link record.** A chart's link names its source (a workbook path with sheet and range or defined
   name, or a dataroom item), a hash of the values read, and a link id. When GridCraft owns the workbook,
   it keeps a hidden defined name `_craft_link_<id>` on the linked block, so inserting rows or columns
   moves the link with the data. DeckCraft looks for that name first. Optional reading rules (transpose,
   which row and column hold the names, extra rows such as reference totals) are part of the record.
   Hidden rows and columns are skipped unless the record says otherwise. Both apps follow the same rules,
   so a hash computed by one matches the other.
3. **CAGR.** For date categories the number of years is `YEARFRAC(start, end, 0)` (US 30/360). For plain
   categories it is the number of steps between them. GridCraft's Insert CAGR and chart CAGR labels use
   the same rule, so the same data gives the same rate everywhere.
4. **Rounding.** The crate's `round_preserving_total` and GridCraft's rounding functions return the same
   results, checked against one shared set of test cases.
5. **Number formats.** The spec describes formats; each app renders them. If GridCraft's number-format
   code becomes a shared crate (a separate proposal), both apps render through it.

## What this RFC decides

1. **The crate lands in craft-libs** with a README, CI and a root README entry. Named maintainer:
   psperera, per RFC-0009 §5.
2. **The first kinds are:**
   - the stacked, clustered and 100% column and bar family;
   - waterfall (stacked and hatched segments, connectors, computed totals);
   - Mekko, with a % axis and with a unit axis;
   - scatter and bubble with quadrant lines and captions;
   - decorations: total labels, connectors, a difference arrow and CAGR labels;
   - rounding that preserves totals.

   Later kinds each get their own issue:
   - combination, line, stacked and 100% area, pie and doughnut, Gantt and timeline;
   - butterfly/tornado, football field, box-and-whisker, candlestick, bar-of-pie and funnel, all built
     from the kinds above;
   - annotations: CAGR arrows, total and level difference arrows, axis breaks and value lines.
3. **Commands** follow each app's naming:
   - in both apps: `insert.businessChart`, `businessChart.data`, `businessChart.kind`,
     `businessChart.labels`, `businessChart.style`, `businessChart.decorations` and
     `businessChart.templates`. The last lists every kind and variant the build supports, with its
     parameter schema;
   - in GridCraft: `businessChart.fromRange` and `businessChart.source`;
   - in DeckCraft: `links.list`, `links.update`, `links.auto`, `links.revert`, `links.relink`,
     `links.clean` and `links.select`.

   They are reachable from the CLI, the control port and MCP, as every command already is.
4. **Template fill.** `deckcraft-cli fill --template T.pptx --data D.json --out O.pptx` fills named
   charts, tables and text in a template from JSON. The JSON schema is DeckCraft's (RFC-0009 §4), and
   the subcommand follows the CLI conventions in RFC-0012.
5. **Built-in templates and colour schemes are original designs**, recorded in `ATTRIBUTION.md` like every
   asset.
6. **Names follow RFC-0029.** Command ids and stored values such as `"kind": "mekko"` are exempt. Each
   user-facing kind name gets a recorded check in the app's `docs/feature-names.md` before it ships. If
   the check doesn't settle "Mekko", the label falls back to a descriptive name such as "variable-width
   chart".

## Acceptance requirements

- **The display list is tested by geometry, never by pixels.** That covers bar extents, label anchors,
  connector endpoints and totals for synthetic specs, plus property tests that every spec within the
  limits lays out without panicking and with bounded output.
- **Round trips use files generated in code**, as both repositories require. A DeckCraft deck holding each
  kind survives `.pptx` save and reopen with an identical spec. A GridCraft workbook does the same in
  `.xlsx`.
- **Every new command passes each app's existing command tests:** `every_command_survives_empty_params` in
  GridCraft, and `every_command_survives_empty_and_hostile_params` and `every_command_id_is_registered_once`
  in DeckCraft.
- **Unsupported kinds say so.** A kind a build doesn't support returns a clear error through every
  interface and isn't listed by `businessChart.templates` (house rule 16).
- **The shared rules have shared tests.** The CAGR and rounding cases run in the crate and in GridCraft, and
  a link hash computed by GridCraft matches DeckCraft's for the same block.

## Landing order

1. craft-libs: the crate, with the column and bar family, rounding and number formats.
2. DeckCraft: the element, its commands, and the `.pptx` tag and `.xlsx` part, on storytold/deckcraft#2135's carrier.
3. GridCraft: `businessChart.fromRange`, in-sheet charts, `.xlsx` persistence, Insert CAGR and the
   rounding functions.
4. Links: DeckCraft's `links.*` commands and Links pane; GridCraft keeps the link names.
5. DeckCraft's datasheet (values only at first) and `deckcraft-cli fill`.
6. Waterfall, Mekko and scatter: one PR per kind in the crate, then one per app.

Each step starts from an issue in the repository it lands in.

## What this RFC does not decide

- **Reading other products' chart data.** Their storage isn't public, so decoding it would break the clean
  room. Charts are rebuilt from their source data instead.
- **Compatibility with presentation add-ins.**
- **Changes to the native chart engines.**
- **Formulas in DeckCraft's datasheet.** They need GridCraft's calculation engine as a shared crate, which
  would be its own proposal. Until then the datasheet holds values.
- **Importing other automation formats into `fill`.** That is a separate DeckCraft issue, if it is
  proposed at all.
- **DeckCraft-only elements and tools**: Harvey balls and checkboxes (storytold/deckcraft#2135), agenda and process-flow
  elements, a slide and element library, automatic layout. They go through DeckCraft issues; icons build
  on storytold/deckcraft#1484.

## Open questions

- **Storing the spec in `.xlsx`.** The prototype keeps it in a hidden defined name beside a native chart.
  Would reviewers prefer a custom XML part or an extension list?
- **Live collaboration (RFC-0020).** Business-chart commands are structured commands: ordered submission
  with the chart as the lock scope. Generated labels aren't edited as text, so nothing goes through the
  text CRDT. Does that match how RFC-0020's contracts PR will classify them?
- **Editing in GridCraft.** "Edit in GridCraft" with live updates needs GridCraft's control port to push
  change events, which it doesn't do today. Is that wanted, or is the datasheet plus the `.xlsx` part
  enough?
- **Names.** The crate name and the command ids are open to review.

Context:
- storytold/deckcraft#881, the request this answers;
- storytold/deckcraft#255 (chart data editor) and storytold/deckcraft#241 (chart workbooks on export): native-chart work this
  builds on and doesn't duplicate;
- storytold/deckcraft#2135: Harvey balls and checkboxes, and the generated-element carrier;
- storytold/deckcraft#263 and storytold/deckcraft#1644, and storytold/gridcraft#777: the control channels the CLI drives;
- storytold/craft-contrib#10 (RFC-0009), RFC-0020, RFC-0022, RFC-0029 and #46 (RFC-0030).
