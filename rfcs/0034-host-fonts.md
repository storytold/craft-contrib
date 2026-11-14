# RFC-0034: Fonts a build's host supplies

Related: storytold/craft-libs#223 (fonts across the suite), storytold/craft-libs#197 (interface fonts on the web),
storytold/craft-libs#358 (craft-web-host), storytold/vectorcraft#4091 (closed: "a shared craft-libs host API first"),
storytold/vectorcraft#7300 (a document's own composite fonts), storytold/vectorcraft#1164 and
storytold/vectorcraft#7177 (asks to add fonts, and Google Fonts, without rebuilding), storytold/vectorcraft#6497
(`text.addFonts`: font files' bytes for the session), storytold/figcraft#1510 (downloading a
missing font from the font store's sources, desktop).

## Summary

A build whose host supplies its fonts, which today means a web build in a browser tab and a
headless engine on a server, gets its document fonts from the page or service that hosts it. Both
are wasm32 builds, where `craft-font-store` already builds its indexes and checks but no store
folder or HTTP client, so no new build switch is needed. The host gives
the build one or more font indexes in craft-font-index format (the format `craft-font-store`
already reads). The build lists those families in its font menus at once and fetches a face, checked
against the index's sha256, only when type in a document uses it. Until the face arrives, or if it
fails, the type is a missing font, shown and retried the way the desktop shows a missing font.
This follows Illustrator's Auto-activate Adobe Fonts: when a file opens and every font it is
missing is available from Adobe Fonts, Illustrator adds them in the background without a
notification; fonts it cannot add are listed in the Missing Fonts dialog to add or replace, and
type in a font it cannot preview is drawn in the default font and highlighted.

## Motivation

- **A web build has no fonts of its own.** It cannot read the system's fonts, and it cannot
  embed the document fonts the desktop builds embed: storytold/craft-libs#223 measures those at about half of the
  VectorCraft and WordCraft downloads. A host page (craft-web-host) or a hosting service usually
  has the fonts its users' documents need, often as a catalogue far larger than any build should
  carry.
- **The suite already has the format.** `craft-font-store` reads craft-font-index indexes: every
  file with its location, size, sha256, style, weight and PostScript names. A host should hand a
  build the same thing, not a second catalogue format per app.
- **Each app would otherwise invent its own.** A fork of VectorCraft's web build has had one since
  October 2026, as a command taking `{family, style, url, sha256}` records. storytold/vectorcraft#4091 was
  closed with a request for a shared contract first. This RFC is that contract.
- **Users ask for it.** storytold/vectorcraft#1164 asks to add a font to a released build, and
  storytold/vectorcraft#7177 for Google Fonts
  inside VectorCraft. On the desktop, FigCraft already downloads a missing font from the font
  store's built-in Google Fonts and Noto indexes (storytold/figcraft#1510), and storytold/vectorcraft#6497 would let a page hand VectorCraft
  font files' bytes. Neither gives a web build a catalogue to fetch from when type uses it.

## Proposal

### 1. What a host gives

1. A host gives a build one or more indexes in craft-font-index format, each as its bytes or as an
   https URL with the sha256 of the index. The build checks them by the same rules
   `Index::from_user_bytes` applies (version, https URL templates, safe paths, sha256s, sizes).
   An index may list a single file, so a host that hands over one face at a time does not have to
   build a catalogue first. (Non-normative: a build that already accepts another form may keep
   accepting it while it moves to indexes.)
2. A host's index is a source of its own kind, `host:<id>`, beside the built-in and `user:`
   sources. Like a user's source it is never curated, and its ID cannot stand in for a built-in
   one.
3. Sources are editor-wide, not per document. A host may add indexes at any time. Adding a list is
   all or nothing: a list the build refuses (malformed, over the build's limit) adds nothing.
4. A build may also accept font files' bytes, as storytold/vectorcraft#6497 proposes (`text.addFonts`), for
   a host that fetched the files itself. Those faces are added for the session, as this RFC's
   fetched faces are.
5. A build reads and checks a host's index with `craft-font-store`'s own types (`Index`,
   `Family::face`, `verify_font`), which build on every target, wasm32 included; only the store
   folder, its HTTP client and language packs are native only. Every build then applies the same
   checks, and none carries a second reader.

### 2. When a face is fetched

1. Never at startup and never for listing. A face is fetched only when type in an open document
   uses it, through a URL the index gives, best first, as `craft-font-store` tries them.
2. A face is found by family and style exactly, as `craft-font-store`'s `Family::face` finds it.
   The build does not fetch another style of the family in place of one the index does not list;
   that style is drawn as the app draws any style it does not have (section 5).
3. Previewing a family in a font menu does not fetch it. A family with a face not yet fetched is
   listed by name, without a sample, until it arrives; then its sample is drawn.

### 3. What a fetched face must be

1. Its bytes match the index's sha256, and its size when the index gives one. In a browser, the fetch carries the sha256 as
   Subresource Integrity, so the browser refuses a mismatch too.
2. The file actually contains the family and style the index says. A file of another family fails
   that source even when the family is otherwise available.
3. A face that fails either check fails its source. Nothing in it is used.

### 4. Activating a face is not an edit

Adding an index and the arrival of a face change no document: no undo step, nothing in a journal or
a saved file, the document's modified state unchanged. Type that uses the face is laid out again
when it arrives, once per batch of arrivals, and never in the middle of an edit in progress.

### 5. Until it arrives, and if it fails

1. A family whose faces have not arrived is not an installed family. Commands that need a loaded
   face (glyph alternates, OpenType features, outlines) treat it as missing; a font menu and the
   command that replaces a font accept it by name.
2. Type in a face that is pending or failed is drawn in the fallback face and marked as a
   substituted font, as the desktop marks a missing one.
3. A failed face is reported as missing where the app reports missing fonts. The user can retry it
   there; a retry fetches it again. A source the host adds again replaces the failed one.
4. A style the index does not list, of a family it does list, is drawn and reported as the app
   draws and reports a style it lacks of an installed family. (VectorCraft draws that family's
   nearest loaded style and reports a substitute; Illustrator draws its default font, highlighted.)

### 6. A host's family never shadows a document's own font

A document can define its own composite font (storytold/vectorcraft#7300). When it shares a name
with a family from a host's index, the document's definition wins in that document, and opening
it never drops its definition.

### 7. Who may add an index

1. The host, and only the host. A host's index arrives through the hosting contract
   (craft-web-host's messages, or the engine's API for a headless host), never from a document.
2. A host adding an index is the consent to fetch from it. A build without a font store has no
   consent switch of its own; whether a user's fonts may come from Google Fonts, say, is the host's
   decision, made in the host's own interface or policy, as the desktop's is made in the font
   store's "Allow downloading fonts?" prompt.
3. An agent, a script or a plug-in running in the build cannot add an index or turn fetching on:
   the build's commands for adding one are refused from its control channel, its MCP server,
   scripts and plug-ins. Setting type in a family a host's index lists is an ordinary edit, and
   fetches that face as any edit would. This is `craft-font-store`'s rule on the desktop, where an
   agent gets `Error::NotAllowed` for adding a source or turning the network on, and may install
   only while the user has downloading on (storytold/figcraft#1510). A host that wants its own agents to
   choose sources gives them a way in its own interface, not the build's.

## What this leaves alone

- The desktop builds' font store, its folder, its consent switch and Craft Font Manager (storytold/craft-libs#223).
  A desktop build may accept a host's index too, but nothing here requires it.
- Interface fonts (storytold/craft-libs#197). They stay embedded.
- Each build's limits: the size of one file, fetches in flight, timeouts, the number of sources.
  Each build states its own.
- How the app's missing-fonts interface looks.
- Where the files are hosted. That is the host's choice, and so is its privacy policy (raised on
  storytold/craft-libs#223 about Google Fonts).

## Acceptance tests

Every build that adopts this RFC passes, from shared fixtures:

1. An index with a wrong sha256, an http URL or an unsafe path is refused, and adds nothing.
2. Listing a family fetches nothing. Opening a document that uses it fetches only the faces its type
   uses, and a style the index does not list fetches nothing.
3. A file whose bytes do not match fails the source; a file of another family fails it.
4. A face's arrival leaves the document unmodified and adds no undo step.
5. A failed face draws in the fallback, is reported missing, and a retry fetches it again.
6. A document whose composite font shares a host family's name keeps its definition through
   open, edit and save.
7. An index added through the control channel, the MCP server, a script or a plug-in is refused
   and adds nothing; setting type in a family of a host's index from any of them fetches the face.

## Reference implementation, non-normative

A fork of VectorCraft's web build implements an earlier form of this (`text.addFontSources` with
`{family, style, url, sha256}` records, and `text.addFont`); it is not merged upstream. Its
November 2026 review brought it to sections 2 to 6, with exceptions: it fetches a family's
nearest listed style for a style no source lists (sections 2.2 and 5.4 say it should not), and it
lacks the size and style checks of section 3. Two browsing paths (turning the font wheel over
families, and selecting a row in Find Font) fetch faces, which section 2.3 rules out. It refuses sources from scripts and plug-ins but not
from its control channel or MCP server, which section 7.3 also refuses. Adopting this RFC would
mean reading craft-font-index indexes in place of that record, dropping that nearest-style fetch,
adding those two checks, fetching nothing while browsing, and refusing those two callers.
The checks on what a fetched face must be, the not-an-edit rule and the
composite-font rule came out of that review.

## Open questions

1. **The face's identity in the document.** storytold/craft-libs#223 notes a document records only family and style.
   Should a host's index be allowed to say which PostScript name a document's family and style
   mean, so a lookalike is not taken for the font?
2. **Which interfaces count a host family as available.** In the fork, Find Font, Recent Fonts and
   font replacement still treat a face not yet fetched as unavailable. Should every interface that
   lists fonts treat a host family as available, as the font menu does?
3. **Names.** A document that names a family by an alias or a PostScript-derived name ("RobotoSlab")
   never matches a host family "Roboto Slab" in the fork. Matching names is a separate question from
   question 1's identity: should a build match a host family by the index's PostScript names too?

## References

- Adobe, Preview, add, or replace missing fonts (Illustrator User Guide): https://helpx.adobe.com/illustrator/desktop/design-with-text/fonts-and-scripts/preview-add-or-replace-missing-fonts.html
- craft-font-index format: https://github.com/aiongg/craft-font-index
- `craft-font-store`: https://github.com/storytold/craft-libs/tree/main/crates/font-store
