# RFC-0086: Shared ACCESSIBILITY.md for Craft applications

Updates: [RFC-0014](0014-accessibility.md)

## Summary

Publish a shared user-facing accessibility statement in
`standards/app-accessibility.md` and copy it into root `ACCESSIBILITY.md` in
every public Craft repository that ships application builds, followed by a short
`## This application` section for repository-specific links and details.

GitHub recognizes `ACCESSIBILITY.md` as a community health file and surfaces it
in an **Accessibility** tab on the repository overview and in the **About**
sidebar. Keeping one canonical source in `craft-contrib`—following the same
shared-prefix and integrity-stamp model as [RFC-0072](0072-shared-agents-md.md)
(`standards/app-agents.md`)—gives users a predictable, plain-language home for
accessibility goals, workarounds, and barrier reporting without letting 18
copies drift apart.

## Motivation

[RFC-0014](0014-accessibility.md) and
[`standards/accessibility.md`](../standards/accessibility.md) establish strong
engineering and review rules for contributors and coding agents, and each
repository's `AGENTS.md` summarizes them. However, a user who arrives at an
application repository after hitting a keyboard, screen-reader, magnifier, or
contrast barrier has no surfaced user-facing statement telling them:

- what assistive technologies and platforms the app targets (`egui` +
  `AccessKit`, WCAG 2.2 AA via WCAG2ICT on desktop and WCAG 2.2 AA on web), and
  what is still early-alpha work in progress;
- known suite-wide barriers and practical workarounds (including `<app>-cli`
  headless workflows where available);
- how to report a barrier without disclosing a disability or recording
  screenshots, and how `accessibility`, `needs screen reader test`, and
  `priority: high` / `priority: critical` triage work.

GitHub now highlights `ACCESSIBILITY.md` directly on repository overview pages
([GitHub Blog announcement](https://github.blog/open-source/maintainers/accessibility-statements-now-have-a-home-on-github/)).
Inheriting a single organization-wide file from `storytold/.github` would also
attach an application accessibility statement to forks, test-corpus
repositories, fonts, and non-UI crates where it does not fit. Maintaining 18
independent hand-written files would drift as shared `AccessKit` and `craft-libs`
capabilities improve. A single reference document in `craft-contrib`, copied
into the application repositories with a small per-app tail, avoids both
problems.

## Proposal

1. **The shared reference document.** `standards/app-accessibility.md` in this
   repository holds the shared user-facing accessibility statement, ending with
   the `## This application` heading. It is a living document under
   [RFC-0013](0013-amending-rfcs.md) point 5: wording fixes and updates to
   factual status (such as newly resolved suite-wide barriers) are ordinary pull
   requests here; policy changes to our accessibility commitments or contributor
   rules require an RFC updating [RFC-0014](0014-accessibility.md) or this RFC.
2. **Scope.** Every public `storytold` Craft repository that publishes
   application builds carries a root `ACCESSIBILITY.md`. Today those 18
   repositories are:
   `anicraft`, `cadcraft`, `craft-anything`, `craft-launcher`, `deckcraft`,
   `designcraft`, `effectcraft`, `figcraft`, `filmcraft`, `gridcraft`,
   `lightcraft`, `materialcraft`, `pdfcraft`, `photocraft`, `soundcraft`,
   `texturecraft`, `vectorcraft`, and `wordcraft`. New Craft applications added
   to the suite include `ACCESSIBILITY.md` when created.
3. **Repository file layout and integrity stamp.** Once this RFC merges, each
   application repository's root `ACCESSIBILITY.md` carries the normalized
   contents of `standards/app-accessibility.md` between HTML comment markers:

   ```markdown
   <!-- craft-accessibility:start source=<full merged commit SHA> sha256=<digest> -->
   ...normalized contents of standards/app-accessibility.md, ending with ## This application...
   <!-- craft-accessibility:end -->
   ```

   HTML comments are hidden when GitHub renders the **Accessibility** tab. Below
   `<!-- craft-accessibility:end -->`, each repository describes the application
   itself: published platforms, its `<app>-cli` headless binary (where shipped),
   app-specific known barriers if any, and direct links to open an issue, view
   open `accessibility` issues, and view pull requests labelled
   `needs screen reader test` in that repository.
4. **Validation and sync.** A follow-up pull request in
   `storytold/craft-common-actions` extends the existing `house-rules` generator
   and check (or adds a companion mode using the same normalization and commit
   ancestry check) to verify `craft-accessibility` blocks in repositories where
   `ACCESSIBILITY.md` is present: an edited block or unmerged revision fails,
   and an intact older revision warns when `standards/app-accessibility.md` has
   changed on `main`.
5. **Amendment to RFC-0014.** Updates [RFC-0014](0014-accessibility.md) point 10
   ("Where it lives") to add `standards/app-accessibility.md` here and root
   `ACCESSIBILITY.md` in application repositories alongside
   `standards/accessibility.md`.

## Rollout

1. Merge `standards/app-accessibility.md` and this RFC in `craft-contrib`.
2. Stamp the draft pull requests across all 18 application repositories from the
   merged `craft-contrib` `main` commit SHA and mark them ready for review:
   - [storytold/anicraft#602](https://github.com/storytold/anicraft/pull/602)
   - [storytold/cadcraft#3430](https://github.com/storytold/cadcraft/pull/3430)
   - [storytold/craft-anything#2377](https://github.com/storytold/craft-anything/pull/2377)
   - [storytold/craft-launcher#1598](https://github.com/storytold/craft-launcher/pull/1598)
   - [storytold/deckcraft#2344](https://github.com/storytold/deckcraft/pull/2344)
   - [storytold/designcraft#3197](https://github.com/storytold/designcraft/pull/3197)
   - [storytold/effectcraft#4140](https://github.com/storytold/effectcraft/pull/4140)
   - [storytold/figcraft#2588](https://github.com/storytold/figcraft/pull/2588)
   - [storytold/filmcraft#5213](https://github.com/storytold/filmcraft/pull/5213)
   - [storytold/gridcraft#4711](https://github.com/storytold/gridcraft/pull/4711)
   - [storytold/lightcraft#4707](https://github.com/storytold/lightcraft/pull/4707)
   - [storytold/materialcraft#315](https://github.com/storytold/materialcraft/pull/315)
   - [storytold/pdfcraft#8184](https://github.com/storytold/pdfcraft/pull/8184)
   - [storytold/photocraft#16614](https://github.com/storytold/photocraft/pull/16614)
   - [storytold/soundcraft#1731](https://github.com/storytold/soundcraft/pull/1731)
   - [storytold/texturecraft#793](https://github.com/storytold/texturecraft/pull/793)
   - [storytold/vectorcraft#9208](https://github.com/storytold/vectorcraft/pull/9208)
   - [storytold/wordcraft#7710](https://github.com/storytold/wordcraft/pull/7710)
3. Add `craft-accessibility` block generation and checking in
   `storytold/craft-common-actions`.

## Alternatives

- **Put `ACCESSIBILITY.md` in `storytold/.github` for org-wide inheritance.**
  One file to maintain, but GitHub would display an `egui` / `AccessKit`
  application accessibility tab on library forks (`egui`, `wgpu`, `winit`,
  `harfbuzz`), corpus repositories, and infrastructure tools, and per-app CLI or
  issue links could not appear in the page.
- **Maintain independent `ACCESSIBILITY.md` files per repository.** Suite-wide
  facts (`AccessKit` platform support, `craft-libs#437` limitations, triage and
  label rules) would quickly drift across 18 repositories.
- **Put only a one-line redirect stub in each repository.** People reading an
  accessibility statement have often just hit a barrier; forcing an extra jump
  to another repository before showing workarounds or how to report the issue
  undercuts the purpose of surfacing `ACCESSIBILITY.md` on the repository page.
