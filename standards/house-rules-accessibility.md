## Accessibility

These apply to native Craft apps and to shared libraries that draw interface. The source is `standards/house-rules-accessibility.md` in `storytold/craft-contrib`. The full rules are [standards/accessibility.md](https://github.com/storytold/craft-contrib/blob/main/standards/accessibility.md).

People use the apps with screen readers, braille displays, magnifiers, high contrast, large text, voice control or only a keyboard. They reach the interface only through its AccessKit tree, keyboard focus and colours. A UI change is not done until it works for them. The bar is WCAG 2.2 AA as applied to software.

1. **Design first.** Name the pattern (menu, tab list, toolbar, tree, grid, list, dialog, slider, canvas) and follow its keyboard model and roles from the WAI-ARIA Authoring Practices Guide and the platform's guidelines. Observe how the reference app exposes it with Accessibility Insights, Accessibility Inspector or Accerciser.
2. **Keyboard.** Every action works from the keyboard, in visual order, with no trap. Esc closes and returns focus. Alt or F10 and F6 behave as on the platform. Move egui's focus (`request_focus`), never only a highlight.
3. **Semantics.** Every control has a role, a translated name that starts with its visible label, and its state and value. Shortcuts and access keys are set on the node, in the current platform's form. A drawn surface publishes nodes for its content, with caret, selection and correct editability.
4. **Feedback.** Announce changes away from focus through a live region.
5. **Seeing.** Contrast at least 4.5:1 for text and 3:1 for controls and focus, in both themes. Never colour alone. Follow the system's high contrast, text size and reduced motion settings.
6. **Pointer.** Anything done by dragging can be done without dragging.
7. **Check the tree, not only the screenshot.** Each new or changed control, dialog or keyboard path gets a kittest test that drives it by keyboard and asserts roles, names, states, focus and announcements. If the helpers do not exist yet, say so in the pull request. Keep `with_accessibility_check` on and CI's platform checks green. With `ui.screenshot`, also read the accessibility tree.
8. **Rule out known failures:** focus left in the document behind a menu, ribbon or KeyTips; editable surfaces reported read-only; dialogs that do not take or return focus, or leave controls behind them reachable; painted rows with no node; identical names; another platform's shortcut text in a name; silent state changes; drag-only actions.
9. **Be honest in the pull request.** Its Accessibility section says what the tests and the tree cover. Tree tests never prove "works with screen readers". Ask for `needs screen reader test` only for an accessibility fix or accessibility logic of its own, and never hold a pull request back for that test.
