# RFC-0014: Accessibility in every UI change

Related: storytold/craft-libs#437 (shared work), RFC-0005, RFC-0006,
RFC-0007.

## Summary

Make accessibility a requirement for every change to an app's interface, as
never-crash is for every change to its code. The craft apps draw their own
interface with egui. People who use a screen reader or braille display, a
magnifier, high contrast, large text, voice control or only a keyboard reach
it only through what each app provides: its AccessKit tree, its keyboard
focus and its colours.

This RFC sets the bar (WCAG 2.2 AA as applied to software), seven rules for
every UI change, a design step, a check of the accessibility tree next to
every screenshot check, a check in CI of what the platform's accessibility
API reports, a list of failures the apps have already produced, two labels,
builds for testers, and a priority rule. This PR publishes the rules as
`standards/accessibility.md`, with a short form that native apps and shared
UI libraries copy into `AGENTS.md`, as they copy the house rules.

Tests and agents can't replace a person using a screen reader, and testers
are few, so no change ever waits for one: a manual test is never a condition
for merging. The aim is that everything a test or an agent can check is
checked, that what only a person can check is easy to find, and that people
who use the apps report the rest.

## Motivation

Accessibility already gets real attention from individual contributors.
AccessKit is on in every app. WordCraft describes its pages to screen readers
(storytold/wordcraft#488). Open PRs bring the ribbon
(storytold/wordcraft#2012) and formatting at the caret
(storytold/wordcraft#4860) to screen readers. EffectCraft is adding an
accessibility audit test (storytold/effectcraft#1335), and PdfCraft's Linux
accessibility is being fixed (storytold/pdfcraft#4612). About two dozen
screen-reader and keyboard issues have been filed. But nothing written asks
for any of it, so the result depends on who picks up which issue:

- **No rule.** No app's `AGENTS.md` asks for accessibility. Neither do the PR
  templates, the `CONTRIBUTING` files, the shared house rules (RFC-0006), or
  RFC-0005 and RFC-0007. Agents follow their instructions literally
  (RFC-0006), so what isn't written isn't done.
- **The check is visual.** `AGENTS.md` tells agents to take `ui.screenshot`
  and compare it with the reference app. Nothing tells them to read the
  accessibility tree, although PdfCraft's `ui.inspect` already returns it.
- **The automatic check covers names only.** egui_kittest fails a test when
  a visible input widget has no accessible name, which is why most controls
  are named. Nothing checks focus, announcements, editability or contrast.
  WordCraft's page passed its tree tests, yet a blind user found that NVDA
  doesn't echo typing: the page is a `Document` node, which AccessKit always
  reports as read-only (storytold/wordcraft#3174). KeyTips left focus in the
  page, so their letters typed into it (storytold/wordcraft#3171). No app
  uses a live region to announce anything, so Ctrl+B is silent; only
  craft-keymap's shortcut editor uses one.
- **The work can't be found.** No app repository has an accessibility label.
  storytold/wordcraft#2012 has been ready since 15 October without a
  screen-reader test, and a report from a JAWS and NVDA user
  (storytold/filmcraft#1074) has no reply yet.
- **The last step has no owner.** Agents are told not to drive the GUI with
  synthetic input on a shared machine, and PRs say so: "No real VoiceOver
  session was run (it needs a person's screen and keyboard)"
  (storytold/lightcraft#3216). No app's CI keeps a build of a PR, so a
  tester has to build it from source.
- **Progress doesn't show it.** WordCraft counts Accessibility as one area
  with weight 1 of 100, mostly Word's tools for making accessible documents.
  GridCraft's gap list says "no accessible grid/ribbon tree".

This is cheap to fix now. Later, every menu, dialog and drawn surface built
without real focus and semantics has to be reworked.

## Proposal

1. **The bar.** The interface meets WCAG 2.2 level AA as applied to software
   by W3C's WCAG2ICT, the approach EN 301 549 (chapter 11) takes for software
   that public bodies buy. The web builds meet WCAG 2.2 AA directly. This
   covers people who use screen readers or braille, magnifiers, high
   contrast, large text, voice control or only a keyboard. Assistive
   technology reaches the apps only through AccessKit (UI Automation on
   Windows, NSAccessibility on macOS, AT-SPI on Linux), so that is what we
   build and test.
2. **Seven rules for every UI change.** WCAG success criteria are in
   parentheses.
   - *Keyboard.* Every action works from the keyboard, in the order things
     appear, with no trap; freehand drawing is the exception WCAG allows. Esc
     closes a popup or dialog and puts focus back where it was. Alt or F10
     moves focus into the menu bar or ribbon, and F6 between regions, where
     the platform does so. Single-key shortcuts can be turned off or
     remapped. (2.1.1, 2.1.2, 2.1.4, 2.4.3)
   - *Real focus.* Keyboard navigation moves egui's focus (`request_focus`),
     never only a painted highlight, so screen readers and magnifiers follow
     it. Focus is visible in both themes and not hidden under other UI.
     (2.4.7, 2.4.11)
   - *Name, role, state.* Every control exposes its role, a translated name
     that starts with its visible label, and its state and value: checked,
     pressed, selected, expanded, disabled. Shortcuts and access keys are set
     on the node, in the current platform's form. (2.5.3, 4.1.2)
   - *Spoken feedback.* A change that happens away from focus, such as a
     toggle by shortcut, "Saved", a result count, an error or a finished
     export, is announced through a live region. Errors say in text what is
     wrong. Messages that close on their own can be read again. (2.2.1,
     3.3.1, 4.1.3)
   - *Drawn surfaces.* A page, canvas, grid, timeline or layer list publishes
     its content as AccessKit nodes: text with caret and selection, cells
     with their position, objects with their alt text, and the caret's place
     on screen for magnifiers. An editable surface is reported as editable.
     (1.1.1, 1.3.1, 4.1.2)
   - *Seeing.* Text has a contrast of at least 4.5:1, and controls and focus
     indicators at least 3:1, in light and dark themes. Colour is never the
     only signal. The app follows the system's high contrast, text size and
     reduced motion settings. (1.4.1, 1.4.3, 1.4.4, 1.4.11)
   - *Pointer.* Anything done by dragging can also be done without dragging,
     unless the drag is the point, as in freehand drawing. Targets are at
     least 24 by 24 points or have that much space around them. (2.5.7,
     2.5.8)
3. **Design before building.** Before writing a UI feature, write down in the
   issue or PR:
   - which standard pattern it is (menu bar, menu, tab list, toolbar, tree,
     grid, list, dialog, slider, canvas), and follow that pattern's keyboard
     model and roles from the WAI-ARIA Authoring Practices Guide and the
     platform's own guidelines;
   - how the reference app, and other apps of the same kind that
     screen-reader users rely on, expose it, observed from outside with the
     platform's inspector (Accessibility Insights for Windows, Accessibility
     Inspector on macOS, Accerciser on Linux), under the existing clean-room
     rules;
   - how a keyboard user, a screen-reader user and a magnifier user do the
     task.

   Record the observed behaviour in the app's parity docs, as WordCraft
   records typing in `docs/typing-parity.md`.
4. **Check the tree, not only the picture.**
   - A change that adds or changes a control, a dialog or a keyboard path
     adds or updates an egui_kittest test. The test drives it by keyboard and
     asserts on the accessibility tree: roles, names, states, where focus is
     after each step, and the announcement if there is one. Until the shared
     helpers for focus and announcements exist (storytold/craft-libs#437,
     part 4), a PR that can't write such a test says so in its Accessibility
     section, and review doesn't block on it.
   - The built-in name check stays on. Turning it off
     (`with_accessibility_check(false)`) needs a comment saying why.
   - Whenever an agent takes `ui.screenshot`, it also reads the
     accessibility tree of the same window and checks it against point 2:
     through the control channel where it returns the tree, otherwise in a
     kittest test.
   - A test checks the contrast of new colours in both themes.
   - **Check what the platform reports, not only the tree.** A correct
     AccessKit tree can still reach screen readers wrong: AccessKit's Windows
     adapter reports every `Document` as read-only (storytold/wordcraft#3174),
     and NVDA finds headings only by a text attribute the adapter doesn't
     set. So CI also reads the running app through the platform's
     accessibility API, with standard tools and no screen reader. On Windows
     that is a UI Automation scan with Axe.Windows (the rules engine of
     Accessibility Insights) and a few UI Automation client checks of what
     screen readers rely on: focus, names, read-only state, the text pattern
     and heading styles. storytold/wordcraft#4951's AccessKit fork already
     tests its text this way. These run as shared jobs in
     craft-common-actions, so contributors never install or run a screen
     reader. macOS and Linux follow when someone sets them up.
5. **Rule out the failures we already have.** Every UI PR checks for:
   - focus left in the document behind a menu, ribbon or KeyTips, so keys
     type into the page (storytold/wordcraft#3171);
   - an editable surface reported as read-only (storytold/wordcraft#3174);
   - a dialog that doesn't take focus, doesn't give it back, or leaves the
     controls behind it reachable (storytold/wordcraft#4269);
   - painted rows, tiles or handles with no node, or with the same name
     (storytold/soundcraft#651, storytold/craft-libs#410);
   - no visible focus (storytold/wordcraft#4495);
   - a state change nobody hears, such as Bold by shortcut;
   - another platform's text in a name, such as "Paste (⌘V)" on Windows
     (storytold/wordcraft#3164);
   - an action that only works by dragging (storytold/filmcraft#1074).

   Failures found later in review or by testers are added to the list in
   `standards/accessibility.md`.
6. **Say what was checked.** A UI PR's description has an Accessibility
   section. It says what the tests and the tree check cover. Tree tests
   alone never support "works with screen readers". When the PR fixes an
   accessibility problem, or builds accessibility logic that standard
   widgets don't give (a drawn surface's tree, its own focus handling, text
   exposed as a document) with a real risk that only a person can judge, the
   section says so, and the triage bot or a triager adds
   `needs screen reader test`. A feature built from standard controls doesn't
   get the label.
7. **Labels and builds for testers.**
   - `accessibility` in every repository, with the description craft-libs
     already uses: "Barrier affecting people with disabilities". The triage
     bot applies it as it applies `ui`.
   - `needs screen reader test` on the PRs point 6 describes, and on fixes
     for problems people reported while using assistive technology. It never
     blocks a merge: the PR merges on its tests and review as usual. The
     label stays until someone tests a build that contains the change and
     reports on the PR what they heard, with the screen reader, its version
     and the platform. If the test fails, the issue is reopened. Everything
     else is tested the way it reaches users: people who use the apps with
     assistive technology report what doesn't work.
   - For PRs with that label, CI keeps a Windows build of the PR, and macOS
     and Linux builds where the gate makes them, through
     craft-common-actions.
   - People who test with assistive technology can get GitHub's Triage role
     to set and clear these labels.
8. **Priority.** This places accessibility barriers on RFC-0007's levels
   without changing the levels. A barrier that stops a keyboard or
   screen-reader user from a core task (open, type, edit, save, reach the
   menus or ribbon, use a dialog) is "a core tool broken or unusable", so
   `priority: high`; if it can cost them their work, `priority: critical`. A
   barrier someone met while using assistive technology is a reproduced
   failure, not a code-reading finding. If RFC-0007 is accepted first, this
   point updates it (`Updates: RFC-0007`, as RFC-0013 proposes); if this RFC
   goes first, RFC-0007 takes the rule into its level definitions.
9. **Measure it.** Each app's progress docs say whether its core tasks can be
   done with a screen reader, with only a keyboard, at 200% text size and in
   high contrast, as part of its readiness. Tools for making accessible
   documents, such as alt text and the accessibility checker, stay a feature
   area.
10. **Where it lives.**
    - The rules, points 1 to 9, are `standards/accessibility.md` in this
      repository: a living document, like the house rules. Failures found
      later are added to its list, and other changes are proposed here for
      maintainer review.
    - Agents may not follow links (RFC-0006), so the short form,
      `standards/house-rules-accessibility.md`, is a generated block of its
      own, as the Rust rules are. Native apps and shared libraries that draw
      interface, such as craft-libs, copy it into `AGENTS.md` after the Rust
      block. It is about 2.9 KB, within the 3 KB that RFC-0006 sets for the
      core block, which has no room left for it. Each app's own section below
      the blocks adds what is specific to it, such as its test harness or its
      drawn surfaces.
    - The house-rules check in craft-common-actions learns the block's name
      and an input that requires it, as it has for the Rust block. Then each
      of those repositories takes the block in its next house-rules update.
    - This PR adds both files and lists them in the README.
    - Shared code (a live-region helper, real focus in craft-menu, the tree
      in every control channel, test helpers, PR builds, platform checks) is
      tracked in storytold/craft-libs#437.
11. **Owner and review.** Like the house rules, the rules belong to the
    maintainers and change through pull requests here. The RFC's author is a
    contact for questions and for screen-reader testing, not a gate.
    Reviewers check the Accessibility section and the tests on UI PRs.
    CodeRabbit, which already reviews every PR, can take the first pass: each
    app's `.coderabbit.yaml` gets a `path_instructions` entry for its UI crate
    that points to the accessibility block in its `AGENTS.md`.

## Alternatives

- **Leave it to each app.** This is the current state: real work, but
  uneven, and the same failures repeat from app to app.
- **An accessibility pass later.** Every surface built without focus and
  semantics would have to be reworked, and agents would keep building new
  ones meanwhile.
- **Automated checks only.** They catch missing names, not whether a task
  can be done. storytold/wordcraft#3174 passed them.
- **No merge without a screen-reader test.** Testers are few and review is
  already the bottleneck (RFC-0008). The label keeps the work findable
  without blocking the queue.
- **The label on every UI PR.** Far more PRs than anyone could test, so the
  label would stop meaning anything. It stays for accessibility fixes and for
  accessibility logic of the app's own.

## Open questions

- Which platform gets PR builds first? Windows has the most screen-reader
  users (NVDA, JAWS, Narrator). Unsigned macOS builds need extra steps to
  open.
- Is 24 by 24 points right for dense toolbars, or should apps rely on
  WCAG's spacing exception there?
