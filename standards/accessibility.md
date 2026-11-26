# Accessibility

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as in RFC 2119 and RFC 8174.

The craft apps draw their own interface with egui. People who use a screen reader or braille display, a magnifier, high contrast, large text, voice control or only a keyboard reach it only through what each app provides: its AccessKit tree, its keyboard focus and its colours. So accessibility applies to every change to an app's interface, as never-crash is for every change to its code ([RFC-0014](../rfcs/0014-accessibility.md)).

These rules apply to native apps and to shared libraries that draw interface. The shared `AGENTS.md` carries a summary of them. Changes are proposed here. Each failure found in review or by testers SHOULD be added to [Known failures](#known-failures).

## The bar

The interface SHOULD meet WCAG 2.2 level AA as applied to software by W3C's WCAG2ICT, the approach EN 301 549 (chapter 11) takes for software that public bodies buy. The web builds SHOULD meet WCAG 2.2 AA directly. This covers people who use screen readers or braille, magnifiers, high contrast, large text, voice control or only a keyboard. Assistive technology reaches the apps only through AccessKit (UI Automation on Windows, NSAccessibility on macOS, AT-SPI on Linux), so that is what we build and test.

## Rules for every UI change

WCAG success criteria are in parentheses.

1. **Keyboard.** Every action SHOULD work from the keyboard, in the order things appear, with no trap; freehand drawing is the exception WCAG allows. Esc SHOULD close a popup or dialog and put focus back where it was. Alt or F10 SHOULD move focus into the menu bar or ribbon, and F6 between regions, where the platform does so. Single-key shortcuts SHOULD be possible to turn off or remap. (2.1.1, 2.1.2, 2.1.4, 2.4.3)
2. **Real focus.** Keyboard navigation SHOULD move egui's focus (`request_focus`) and SHOULD NOT only paint a highlight, so screen readers and magnifiers follow it. Focus SHOULD be visible in both themes and SHOULD NOT be hidden under other UI. (2.4.7, 2.4.11)
3. **Name, role, state.** Every control SHOULD expose its role, a translated name that starts with its visible label, and its state and value: checked, pressed, selected, expanded, disabled. Shortcuts and access keys SHOULD be set on the node, in the current platform's form. (2.5.3, 4.1.2)
4. **Spoken feedback.** A change that happens away from focus, such as a toggle by shortcut, "Saved", a result count, an error or a finished export, SHOULD be announced through a live region. Errors SHOULD say in text what is wrong. Messages that close on their own SHOULD be possible to read again. (2.2.1, 3.3.1, 4.1.3)
5. **Drawn surfaces.** A page, canvas, grid, timeline or layer list SHOULD publish its content as AccessKit nodes: text with caret and selection, cells with their position, objects with their alt text, and the caret's place on screen for magnifiers. An editable surface SHOULD be reported as editable. (1.1.1, 1.3.1, 4.1.2)
6. **Seeing.** Text SHOULD have a contrast of at least 4.5:1, and controls and focus indicators at least 3:1, in light and dark themes. Colour SHOULD NOT be the only signal. The app SHOULD follow the system's high contrast, text size and reduced motion settings. (1.4.1, 1.4.3, 1.4.4, 1.4.11)
7. **Pointer.** Anything done by dragging SHOULD also be possible without dragging, unless the drag is the point, as in freehand drawing. Targets SHOULD be at least 24 by 24 points or have that much space around them. (2.5.7, 2.5.8)

## Design before building

Before writing a UI feature, you SHOULD write down in the issue or pull request:

- which standard pattern it is (menu bar, menu, tab list, toolbar, tree, grid, list, dialog, slider, canvas), and follow that pattern's keyboard model and roles from the WAI-ARIA Authoring Practices Guide and the platform's own guidelines;
- how the reference app, and other apps of the same kind that screen-reader users rely on, expose it, observed from outside with the platform's inspector (Accessibility Insights for Windows, Accessibility Inspector on macOS, Accerciser on Linux), under the clean-room rule;
- how a keyboard user, a screen-reader user and a magnifier user do the task.

You SHOULD record the observed behaviour in the app's parity docs, as WordCraft records typing in `docs/typing-parity.md`.

## Checks

- Where a test is warranted for keyboard or screen-reader behaviour (judged by [testing.md](testing.md)), it SHOULD be an egui_kittest test that drives it by keyboard and checks the accessibility tree: roles, names, states, where focus is after each step, and the announcement if there is one. A new or changed keyboard path, focus move or announcement warrants a kittest test; a control's role or name alone doesn't, because the built-in check covers names. Until the shared helpers for focus and announcements exist (storytold/craft-libs#437, part 4), a pull request that can't write such a test SHOULD say so in its Accessibility section, and review doesn't block on it.
- The built-in name check SHOULD stay on. A test that turns it off (`with_accessibility_check(false)`) SHOULD have a comment saying why.
- Whenever an agent takes `ui.screenshot`, it SHOULD also read the accessibility tree of the same window and check it against the rules above: through the control channel where it returns the tree, otherwise in a kittest test.
- A test SHOULD check the contrast of new colours in both themes.
- CI SHOULD check what the platform reports, not only the tree. A correct AccessKit tree can still reach screen readers wrong: AccessKit's Windows adapter reports every `Document` as read-only (storytold/wordcraft#3174), and NVDA finds headings only by a text attribute the adapter doesn't set. So CI reads the running app through the platform's accessibility API, with standard tools and no screen reader. On Windows that is a UI Automation scan with Axe.Windows (the rules engine of Accessibility Insights) and a few UI Automation client checks of what screen readers rely on: focus, names, read-only state, the text pattern and heading styles. These run as shared jobs in craft-common-actions, so contributors never install or run a screen reader. macOS and Linux follow when someone sets them up.

## Known failures

Every UI pull request SHOULD check for:

- focus left in the document behind a menu, ribbon or KeyTips, so keys type into the page (storytold/wordcraft#3171);
- an editable surface reported as read-only (storytold/wordcraft#3174);
- a dialog that doesn't take focus, doesn't give it back, or leaves the controls behind it reachable (storytold/wordcraft#4269);
- painted rows, tiles or handles with no node, or with the same name (storytold/soundcraft#651, storytold/craft-libs#410);
- no visible focus (storytold/wordcraft#4495);
- a state change nobody hears, such as Bold by shortcut;
- another platform's text in a name, such as "Paste (⌘V)" on Windows (storytold/wordcraft#3164);
- an action that only works by dragging (storytold/filmcraft#1074).

## Pull requests

A UI pull request's description SHOULD have an Accessibility section. It SHOULD say what the tests and the tree check cover. Tree tests alone do not support "works with screen readers". When the pull request fixes an accessibility problem, or builds accessibility logic that standard widgets don't give (a drawn surface's tree, its own focus handling, text exposed as a document) with a real risk that only a person can judge, the section SHOULD say so, and the triage bot or a triager SHOULD add `needs screen reader test`. A feature built from standard controls doesn't get the label.

## Labels and test builds

- `accessibility` is in every repository, with the description "Barrier affecting people with disabilities". The triage bot applies it as it applies `ui`.
- `needs screen reader test` goes on the pull requests described above, and on fixes for problems people reported while using assistive technology. It does not block a merge: the pull request merges on its tests and review as usual. The label stays until someone tests a build that contains the change and reports on the pull request what they heard, with the screen reader, its version and the platform. If the test fails, the issue is reopened. Everything else is tested the way it reaches users: people who use the apps with assistive technology report what doesn't work.
- For pull requests with that label, CI keeps a Windows build of the pull request, and macOS and Linux builds where the gate makes them, through craft-common-actions.
- People who test with assistive technology can get GitHub's Triage role to set and clear these labels.

## Priority

Accessibility barriers sit on the priority levels of RFC-0007 without changing them. A barrier that stops a keyboard or screen-reader user from a core task (open, type, edit, save, reach the menus or ribbon, use a dialog) is "a core tool broken or unusable", so `priority: high`; if it can cost them their work, `priority: critical`. A barrier someone met while using assistive technology is a reproduced failure, not a code-reading finding.

## Progress

Each app's progress docs SHOULD say whether its core tasks can be done with a screen reader, with only a keyboard, at 200% text size and in high contrast, as part of its readiness. Tools for making accessible documents, such as alt text and the accessibility checker, stay a feature area.
