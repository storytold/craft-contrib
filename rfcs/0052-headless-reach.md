# RFC-0052: Parity reports say what a headless client can reach

Each app's parity report compares its engine command registry with the
incumbent app's menus. It doesn't say which features a headless client can
reach: `<app>-cli`, the app's MCP server, or a hosted connector that runs
them. Someone using an app from ChatGPT or Claude can't tell in advance
whether a desktop feature has a tool. This RFC adds one column to the
parity reports that answers that.

The problem, the SoundCraft finding and the table of what BridgeCraft Studio
hides are as reported by Adithya Shankaran (adikshan11) in issue #24, where
they published the connector manifest. This RFC turns their proposal into a
decision. The findings have not been checked against the app repositories;
marking the rows, below, is how each one gets checked.

## Where it comes from

All of this is as reported in adikshan11's issue #24. It has not been
verified against the app repositories.

- Reported in #24: SoundCraft's `docs/parity-checklist.md` shows AudioSuite
  at 0 of 28 menu items in the engine. The report notes that UI-layer
  commands such as windows and AudioSuite are counted by the desktop app's
  own report (`engine.parity` over the control channel). If so, whatever the
  desktop app does there, `soundcraft-cli` and its MCP server can't. The
  other apps haven't been checked for features that exist only in their UI
  layer.
- Reported in #24: a hosted connector removes more on top of that.
  BridgeCraft Studio hosts eight apps and hides or refuses the tools listed
  in its
  [`craft_apps.json`](https://github.com/adikshan11/artcraft-plugin/blob/main/craft_apps.json)
  (WordCraft, DeckCraft, PdfCraft, PhotoCraft, VectorCraft, DesignCraft,
  FigCraft, SoundCraft).
- Reported in #24: most of the hidden tools drive a desktop window that a
  server doesn't have, so nothing is lost by hiding them. The real losses
  are printing and OS-keychain or Windows certificate signing in PdfCraft,
  PDF JavaScript in PdfCraft, live collaboration, library publishing and
  plug-ins in FigCraft, and choosing a record folder in SoundCraft.

The two kinds of gap need different answers. A feature that needs a window
should be marked and left alone. A feature that only lacks an engine command
should be easy to find, so it can move into the engine.

## Decision

- **A `headless:` value on every parity row.** It says whether a headless
  client can do the feature:
  - `yes`: an engine command exists, so `<app>-cli` and the MCP server
    reach it.
  - `no`: only the desktop app can do it.
  - `partial`: some of the feature is reachable. The body says which part.
  - `unknown`: not yet checked. This is the starting value, so no app is
    blocked on auditing every row before it adopts this.
- **A `no` or `partial` row says why**, with one of these in a `reach:`
  field:
  - `window`: the feature drives a desktop window, dialog or pointer.
    Nothing is lost on a server.
  - `os`: the feature needs an OS service the engine doesn't wrap, such as
    printing or a keychain.
  - `engine-gap`: nothing stops it being an engine command. It only isn't
    one yet. These are the rows to move into the engine.
- **Where it lives.** The column is part of the app's parity table. Once
  an app keeps its parity rows as files ([RFC-0044](0044-gap-parity-items.md)),
  the value is a front-matter field of the item file, beside the app's other
  column fields, and the joiner adds the column to the table. An app that
  isn't on RFC-0044 yet adds the column to its parity table by hand, and
  moves it to the item files when it adopts RFC-0044.
- **`cargo xtask parity` fills in what it can.** A row whose command id is
  in the engine's command registry, the one `<app>-cli` and the MCP server
  load, is `yes`. A row that is live only through the desktop app's own
  handlers is not `yes`: it keeps the value written in the file, or
  `unknown`. A row marked `yes` whose id isn't in the engine registry fails
  the check, so the column can't claim a reach the engine doesn't have.
  The test, and this wording, are @hectorviov's. A parity report's "live"
  doesn't mean "in the engine registry": in PhotoCraft it also counts a
  dozen desktop-only sources (`is_live` in `crates/ui-egui/src/menus/items.rs`),
  so filling `yes` from that check would mark all 62 Window items reachable
  from `photocraft-cli` and the MCP server.
- **The summary line counts it.** Each report adds, next to its existing
  totals, how many rows are `yes`, `no` (by `reach:`), `partial` and
  `unknown`.
- **A connector reads the column.** A hosted connector's list of hidden
  tools and refused commands stays in the connector's own manifest. The
  parity column is what tells it, and its users, which hidden items are
  `window` (expected) and which are `os` or `engine-gap` (a real loss).
  What a connector hides is its own decision, and RFC-0016 (a hosted
  connector, [open PR #23](https://github.com/storytold/craft-contrib/pull/23))
  is where that belongs. That part waits on RFC-0016; this RFC does not
  depend on it to be accepted, because the column is useful to people using
  `<app>-cli` and the MCP server without any connector.

## What this asks of each app

- Add the column to the parity report, every row starting at `unknown`.
- Mark rows as they are checked. Rows already known to be UI-layer only
  (SoundCraft AudioSuite) can be marked in the same change.
- Teach `cargo xtask parity` to fill `yes` from the registry and to reject
  a `yes` row with no matching command.
- Tell agents about the column in `AGENTS.md`
  ([RFC-0006](0006-coding-agents.md)).

## Not in this RFC

- Checking the other apps for features that exist only in their UI layer.
  That is the work of marking rows, one app at a time.
- Moving `engine-gap` rows into the engine. Each is its own change in its
  own app.
- What a hosted connector hides or refuses, and how it shows the list to
  users.

## References

- Issue #24 by adikshan11, which this RFC answers; the findings above are as
  reported there, not verified
- @hectorviov's review comment on this RFC's pull request (#52), which gives
  the registry test for `cargo xtask parity`
- [RFC-0015](0015-confined-file-access.md) (confined file access)
- RFC-0016 (a hosted connector), [open PR #23](https://github.com/storytold/craft-contrib/pull/23)
- [RFC-0044](0044-gap-parity-items.md) (gap and parity items are files)
- SoundCraft `docs/parity-checklist.md`
