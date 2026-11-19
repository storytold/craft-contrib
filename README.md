[![Scanned by SonarCloud](https://img.shields.io/badge/Scanned_by-SonarCloud-4b9fd5?style=flat-square&logo=sonarqubecloud&logoColor=white)](https://sonarcloud.io/project/overview?id=storytold_craft-contrib)

# craft-contrib

The Craft apps — PhotoCraft, WordCraft, EffectCraft, VectorCraft and the rest — share one set of rules and one way to make decisions together. This repository holds both. It contains no application code.

**Anyone can take part.** You do not need a special role, an invitation, or a merged pull request. Comments from users, developers and newcomers are all welcome — they are the point of this repository.

## What are the Craft apps?

The Craft apps are a suite of independent creative applications for desktop and web, written in Rust. Each app works on its own, but they share libraries, tools and rules. AI-assisted contributions, including coding agents, are welcome; the shared [AGENTS.md](AGENTS.md) holds the rules that apply to them ([RFC-0048](rfcs/0048-shared-agents-md.md)).

## What lives in this repository

| Folder | What it is |
| --- | --- |
| `rfcs/` | Decision documents. Each one proposes a change and records why it was accepted. |
| `AGENTS.md` | The shared rules for agents, copied into every Craft repository's `AGENTS.md`. |
| `standards/` | The detailed standards that the shared rules link. |

The most important documents:

- [RFC-0000: How we make decisions](rfcs/0000-decision-process.md) — the decision process itself
- [RFC-0013: Amending an accepted RFC](rfcs/0013-amending-rfcs.md) — how accepted decisions are changed
- [AGENTS.md](AGENTS.md) — the shared rules for every Craft repository ([RFC-0048](rfcs/0048-shared-agents-md.md))
- [Claims](standards/claims.md) — claiming issues and duplicate pull requests
- [Pull requests](standards/pull-requests.md) — shape, description and review responses
- [Clean room](standards/clean-room.md) — where knowledge, assets and fixtures may come from
- [Licences](standards/licences.md) — the licence allowlist and our own licence
- [Accessibility](standards/accessibility.md) — the full accessibility standard
- [Time](standards/time.md) — calendar dates and range extension
- [Naming](standards/naming.md) — the rules for other companies' names ([RFC-0029](rfcs/0029-other-companies-names.md))

Each repository copies the shared `AGENTS.md` into its own, between markers, and adds a `## This repository` section about the project. Until a repository carries the shared copy, its `AGENTS.md` still holds the older house rules from [house-rules-core.md](standards/house-rules-core.md), [house-rules-rust.md](standards/house-rules-rust.md) and [house-rules-accessibility.md](standards/house-rules-accessibility.md).

## How a decision is made (the short version)

1. Anyone opens a pull request that adds a file like `rfcs/NNNN-short-title.md`. This is called an **RFC** (Request for Comments).
2. A maintainer starts a **comment period** by posting its end date on the pull request: usually **3 days**, or **7 days** for big decisions (adding or removing an app, or changing the process itself).
3. Everyone comments. This is where the real work happens.
4. After the period, the RFC merges when a maintainer (not the author) approves it and no maintainer's objection is left unresolved.

The full text is in [RFC-0000](rfcs/0000-decision-process.md).

## What is being discussed right now

See the [open pull requests](https://github.com/storytold/craft-contrib/pulls) for proposals currently under discussion. Accepted decisions are recorded in [`rfcs/`](rfcs/).

## How you can help

**Comment on an open RFC.** This helps the most, and it is the easiest thing to do. A useful comment does not need to be technical:

- "I use PhotoCraft daily. This change would break my workflow because …"
- "I tried something similar in my project. What worked was …"
- "I don't understand this part. Could someone explain it?"

Questions are comments too. If an RFC is hard to understand, saying so is valuable — it means the text needs work.

**Open an RFC.** If you want to change a rule or decision, add `rfcs/NNNN-short-title.md` in a pull request. Pick the next free number, describe the problem first and the solution second, and say which apps it affects. An accepted RFC is not edited to change its meaning: add `Updates: RFC-NNNN` or `Obsoletes: RFC-NNNN` below the new RFC's title, as described in [RFC-0013](rfcs/0013-amending-rfcs.md).

**Fix small things.** Typos, broken links and wording clarifications that do not change a rule's meaning can be ordinary pull requests. They still need a maintainer's approval.

## Glossary

| Term | Meaning |
| --- | --- |
| **RFC** | A proposal discussed on its pull request until a decision is made. |
| **Comment period** | The days when a proposal is open for comments. A maintainer starts it by posting the end date. |
| **Maintainer** | A member of the `craft-app-team` GitHub team. Maintainers review and merge RFCs. |
| **Owner** | An admin of the storytold organisation who can decide objections. |
| **Shared rules** | Rules that apply in every Craft repository. They live in `AGENTS.md` here and are copied into each repository's `AGENTS.md`. |
| **`AGENTS.md`** | The instruction file that coding agents read in every repository. |
| **`craft-libs`** | The shared Rust library repository that several apps depend on. |
| **Claim** | Saying on an issue that you are working on it, so two people do not fix the same bug ([RFC-0003](rfcs/0003-claiming-issues.md)). |
| **Triager** | Someone with the GitHub Triage role in one app's repository: labelling and closing issues. |
