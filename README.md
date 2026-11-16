# craft-contrib

<a href="https://sonarcloud.io/project/overview?id=storytold_craft-contrib"><img alt="Scanned by SonarCloud" src="https://img.shields.io/badge/Scanned_by-SonarCloud-4b9fd5?style=flat-square&logo=sonarqubecloud&logoColor=white"></a>

Shared contributor and coding-agent guidelines for the storytold craft apps.
Changes go through RFCs: see [RFC-0000](rfcs/0000-decision-process.md).

The rules copied into each repository's `AGENTS.md` live in
[standards/house-rules-core.md](standards/house-rules-core.md) and
[standards/house-rules-rust.md](standards/house-rules-rust.md)
([RFC-0006](rfcs/0006-coding-agents.md)). Repositories that draw interface
also copy
[standards/house-rules-accessibility.md](standards/house-rules-accessibility.md),
the short form of [standards/accessibility.md](standards/accessibility.md)
([RFC-0014](rfcs/0014-accessibility.md)). The calendar exception is
[standards/time.md](standards/time.md). The rules for other companies' names are
[standards/naming.md](standards/naming.md)
([RFC-0029](rfcs/0029-other-companies-names.md)). Propose a change to those files here.

## Apple proposals under review

[RFC-0041](rfcs/0041-native-apple-shells.md) proposes a scoped Swift/Rust native
host exception, coordinated with the existing mobile and shared-library proposals.
[RFC-0042](rfcs/0042-apple-distribution-lanes.md) separately proposes opt-in Apple
validation/distribution lanes and Mac App Store readiness gates. Neither is adopted
policy before maintainer review; no app instruction blocks or releases change here.
