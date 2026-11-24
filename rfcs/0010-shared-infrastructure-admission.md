# RFC-0010: Shared infrastructure admission and ownership

Related: storytold/craft-libs#24, storytold/craft-libs#96.

## Summary

This RFC establishes the admission criteria and ownership boundary for shared crates in `storytold/craft-libs`. Shared crates must normally be required by at least two independent Craft applications, with an explicit exception mechanism for foundational capabilities such as security boundaries, authentication contracts, interoperability primitives, and common platform contracts. Crates admitted to `craft-libs` must not depend on application-specific crates and must identify a responsible maintainer or maintaining team.

## Motivation

As the Craft ecosystem grows across independent applications, code sharing must be balanced against premature centralization and tight coupling. Extracting single-application utilities into `craft-libs` increases maintenance burden and creates shared crates that lack genuine multi-application demand.

Conversely, an absolute two-consumer rule is too rigid. Some capabilities warrant shared ownership from the beginning, particularly security boundaries, authentication contracts, and cross-application interoperability primitives. Developing these independently in separate application trees risks divergent security models and incompatible data formats.

Additionally, admitting code to shared infrastructure without identifying who maintains it creates unowned crates, lagging compatibility, and delayed releases. Establishing two consumers as the default qualification threshold, combined with documented exceptions for foundational capabilities, strict boundaries against application-specific dependencies, and clear ownership, keeps `craft-libs` focused and sustainable.

## Proposal

1. **Default Two-Consumer Qualification Threshold**: A crate is normally admitted to `craft-libs` as shared infrastructure when it is actively required and consumed by at least two independent Craft applications. Single-application utilities must remain in their respective repository trees until concrete reuse across applications emerges.

2. **Documented Exceptions for Foundational Capabilities**: A crate may be admitted to `craft-libs` with a single initial consumer when the proposal provides explicit documentation demonstrating that the capability meets one of the following criteria:
   - *Security boundaries*: Cryptographic operations, secret handling, or sandbox isolation where centralized review and maintenance are critical.
   - *Authentication contracts*: Identity verification, session tokens, or credential management shared across ecosystem services.
   - *Interoperability primitives*: Core document interchange structures, standard file format parsers, or wire protocol definitions intended for cross-application communication.
   - *Common platform contracts*: Low-level operating system abstractions, thread lifecycle coordination, or native window event primitives that must remain uniform across applications.
   The proposal must explain which cross-app contract would be at risk if implementations diverged, and why that contract needs shared ownership before a second app adopts it. A faster build or a smaller app repository, by itself, is insufficient. For example, a helper that rearranges one app’s export dialog stays in that app until a second app needs it.

   A maintainer other than the proposer approves the exception in the RFC or pull request. The crate README records who approved it, why, and when to re-review the exception if a second consumer has yet to adopt it. At that milestone the maintainer review records whether to retain the exception, move the crate back to the app, or plan a deprecation with its consumers.

3. **Zero App Code Invariant**: Crates in `craft-libs` must never depend on any application crate (such as `photocraft`, `vectorcraft`, or `filmcraft`). Shared crates must remain reusable across applications and must not depend on application-specific crates. Shared presentation components are permitted when their interfaces and behavior are sufficiently generalizable.

4. **Distinction Between Qualification and Implementation Placement**: Qualifying functionality as shared infrastructure does not require moving all related application code into `craft-libs`. Domain workflows, application-specific styling, localized strings, and command dispatchers remain in application repositories. Only reusable data models, algorithms, presentation primitives, and contracts reside in `craft-libs`.

5. **Responsible Maintainer Identification**: Each crate admitted to `craft-libs` must identify a responsible maintainer or maintaining team. This includes explicit responsibility for tracking downstream compatibility, managing crate releases, and responding to security updates.

6. **Shared Crate Lifecycle**:
   - *Identification*: A contributor documents overlapping functionality in two or more applications, or proposes a foundational capability under the exception criteria.
   - *Contract and ownership definition*: The contributor proposes the crate scope, public API types, error structures, and identified maintainer.
   - *Implementation*: The crate is implemented in `craft-libs` with independent unit tests and without application dependencies.
   - *Consumer adoption*: Applications update their dependencies to consume the shared crate, retiring redundant local implementations.
   - *Ownership handoff*: When the responsible maintainer or team steps down, record the vacancy in the crate README and bring it to the craft-libs maintainers. Existing users can keep their pins. Pause new adoption until someone accepts ownership or maintainers record another disposition, including any security maintenance commitments.

## Alternatives

- **Strict two-consumer rule without exceptions**: Prevents premature extraction, but forces security boundaries and interoperability formats to be duplicated until a second application is ready to adopt them.
- **Admit crates on single-application request without justification**: Leads to shared repository bloat, orphaned crates, and unnecessary coupling between `craft-libs` and individual application priorities.
- **Admission without designated maintainer**: Leaves shared crates without ongoing stewardship, causing dependency bit-rot and blocked consumer upgrades.
- **Permit shared crates to depend on application utilities**: Introduces circular dependency risks and prevents applications from evolving their internal architectures independently.

## Open questions

- Should the suite set a default re-review interval for exceptions, alongside the milestone recorded at admission?
- Should the suite set a standing security maintenance policy for crates without an owner?
- Should large file format parsers (such as PDF or PSD) be hosted directly in `craft-libs` or in dedicated standalone repositories under the `storytold` organization?
