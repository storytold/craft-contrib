# RFC-0001: Expose the command catalogue to the web build (`window.artcraft`)

## Summary

Every Craft app exposes its whole command engine to agents through
`<app>-cli mcp`. The **web** build compiles the same engine but exposes no
JavaScript API, so an agent driving the browser tab can only click pixels. This
RFC asks for a small, opt-in JS façade over the existing command dispatcher, and
explains why it is the only way to drive menus by name.

## Motivation

The field trial drove all 13 apps through a page-injected bridge
(keyboard/pointer/canvas capture). It created and exported real files
(PhotoCraft `.psd`/`.png`, VectorCraft, FilmCraft, CADCraft `.dxf`) but found
three things the UI level cannot do:

1. **Menus by name.** Every app renders menus, dialogs and the ribbon **inside a
   full-window canvas** (`ui="canvas"`, `domCount<3`). A page script cannot read
   a canvas, so `{menu:["File","Export","Quick Export as PNG"]}` cannot be
   resolved — only brittle pixel coordinates work (`52,20 / 86,386 / 370,386`).
2. **Reading the document.** No way to ask "what is selected / what is the
   current document", so an agent guesses from a screenshot.
3. **Reliable pixels.** `canvas.toDataURL()` intermittently returns a blank
   frame (WebGPU without `preserveDrawingBuffer`), so screenshots silently fail.

All three disappear with an exact command surface.

## Proposal

Publish, after the engine starts, a JS object on the page:

```js
window.artcraft = {
  commandList(): string,                     // JSON, identical to the CLI's command_list
  commandRun(id: string, paramsJson: string): Promise<string>,
  commandBatch(stepsJson: string): Promise<string>,
  docInspect(): string,                      // JSON
  renderPreview(page: number, maxSide: number): Promise<Uint8Array>, // PNG bytes
};
```

- `commandList()` returns what `list_commands` already returns over MCP: each
  command's **id, label, ribbon/menu path and shortcuts**. That is enough for a
  client to map `"File ▸ Export ▸ Quick Export as PNG"` → id → `commandRun`.
  No new menu model is needed.
- `renderPreview` returns real pixels from the engine, so a screenshot never
  races the swap chain.
- The wire format is byte-identical to the CLI MCP, so one client config works
  across the suite and an in-page bridge is a thin relay.

### Optional: `menuList()`

If the catalogue's paths are not a complete menu tree (enabled/disabled state,
checkable items, submenus that exist as UI but not as commands), add:

```js
menuList(): string   // [{ path: ["File","Export"], id: "file.export.quick.png", enabled: true }]
```

## Sketch

`apps/<app>-web/src/bridge.rs` (new) wraps the existing dispatcher, which the
CLI's `mcp` module already drives; `apps/<app>-web/src/main.rs` publishes it:

```rust
#[wasm_bindgen]
impl Bridge {
    pub fn command_list(&self) -> String { /* serde_json */ }
    pub async fn command_run(&self, id: String, params_json: String) -> Result<String, JsValue> { /* … */ }
    pub fn doc_inspect(&self) -> String { /* … */ }
    pub async fn render_preview(&self, page: u32, max_side: u32) -> Result<Vec<u8>, JsValue> { /* … */ }
}
```

Publish once at startup: `window.artcraft = artcraft_bridge();`.

## Alternatives considered

- **OCR the canvas to find menu labels.** Rejected: slow, locale/theme/scale
  fragile, and the exact catalogue already exists.
- **Keep telling agents to click coordinates.** Rejected: breaks on any layout
  or window-size change; the trial's macros were tied to three fixed pixels.
- **A server-side headless engine.** Rejected: the goal is agent + static host,
  with the server doing nothing.

## Scope / rollout

1. One app first (`pdfcraft` or `gridcraft` — smallest, already have
   `docs/mcp.md` and a `list_commands` tool).
2. A shared helper in `craft-libs` (the catalogue is already common) so each
   app's `bridge.rs` is a few lines.
3. Gate behind a build feature so the default release size is unchanged.

## Security

- The object exists only in the page; only code in that page can call it. The
  intended consumer is a loopback bridge (`127.0.0.1`, a secure context).
- No new network egress; no `eval`; file access is unchanged from what the app
  already does through the browser, and the CLI's
  `--automation-read-root`/`--automation-write-root` confinement can be reused.

## Migration: the bridge's tools today → `window.artcraft`

The loopback bridge's tool names do not change, so existing client configs keep
working; only the implementation behind each tool changes.

| bridge tool today | becomes | why |
|---|---|---|
| `screenshot` / `capture` | `renderPreview(page, maxSide)` | real pixels from the engine; no blank-frame race, no `force:true` |
| `keys_macro {menu:[…]}` | `commandList()` → label/id lookup → `commandRun(id, params)` | **the whole point**: menus by name |
| `keys_macro {find:{text}}` | same | no DOM lookup needed |
| `shortcut` / `type_text` / `pointer` | unchanged | still the way to enter text and make canvas gestures |
| `page_info` / `ui_state` | `docInspect()` (plus the existing UI state) | ask the document instead of guessing from a screenshot |
| multi-step macros | `commandBatch(stepsJson)` | ordered, atomic |
| `insert_file` (drag/drop) | unchanged, plus file commands via `commandRun` | the browser has no paths; the picker/drop stays |
| `open_app` | unchanged | it is only a URL |

## What the trial hit per app (scope)

Observations from the October 2026 trial, to order the rollout. Every app has a
full command catalogue; only the web build lacks the JS surface.

| app | docs | notes |
|---|---|---|
| photocraft | `docs/mcp.md` | exports real `.psd`/`.png` in the browser; parts-packaged web build |
| vectorcraft | `docs/mcp.md` | "Save As" carries a format dropdown |
| filmcraft | `docs/agents.md` | saves a `.fcproj`; MCP has bridge and headless modes |
| lightcraft | `docs/mcp.md` | the library lives in browser storage (`File ▸ Back Up Library`) |
| pdfcraft | `docs/mcp.md` | exporting goes through a file picker |
| effectcraft | `docs/mcp.md` | "GPU device lost" after several WebGPU tabs are opened |
| designcraft | `docs/mcp.md` | edits kept in browser storage |
| figcraft | `docs/mcp.md` | edits kept in browser storage |
| wordcraft | `docs/mcp.md` | edits kept in browser storage |
| soundcraft | `docs/mcp.md` | edits kept in browser storage |
| cadcraft | `docs/mcp.md` | its command line works from typed keys and saves `.dxf` |
| gridcraft | `docs/mcp.md` | saves through a dialog |
| deckcraft | `docs/mcp.md` | (separate bug) the web build panics at startup on wasm32 |
| launcher | - | a catalogue; its download links are external |

Start with **pdfcraft** or **gridcraft**: the smallest apps, and their
`list_commands` already returns each command's label and ribbon path.

## Unresolved questions

- Should `commandList()` be capped/filtered (the catalogue is large)?
- Is `menuList()` needed, or are catalogue paths sufficient for every app?
- Where should the JS surface live — a new `*-web` export, or the boot module?

## Interim

`artcraft.siit.tu.ac.th/experiment1/` shows the transport working today
(SSE bridge, 22 tools, cross-app handoff) with UI-level tools only, so nothing
there needs re-writing when this lands — the same tool names map onto the new
exports.
