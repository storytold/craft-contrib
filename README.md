[![Scanned by SonarCloud](https://img.shields.io/badge/Scanned_by-SonarCloud-4b9fd5?style=flat-square&logo=sonarqubecloud&logoColor=white)](https://sonarcloud.io/project/overview?id=storytold_craft-contrib)

# craft-contrib

The Craft apps — PhotoCraft, WordCraft, EffectCraft, VectorCraft and the rest — share one set of rules and one way to make decisions together. This repository holds both. It contains no application code.

**Anyone can take part.** You do not need a special role, an invitation, or a merged pull request. Comments from users, developers and newcomers are all welcome — they are the point of this repository.

## What are the Craft apps?

The Craft apps are a suite of independent creative applications for desktop and web, written in Rust. Each app works on its own, but they share libraries, tools and rules. AI-assisted contributions, including coding agents, are explicitly welcome — the rules below exist so that this works at scale.

## What lives in this repository

| Folder | What it is |
| --- | --- |
| `rfcs/` | Decision documents. Each one proposes a change and records why it was accepted. |
| `standards/` | The actual rule texts. These get copied into every app's `AGENTS.md`, so coding agents read them. |

The most important documents:

- [RFC-0000: How we make decisions](rfcs/0000-decision-process.md) — the decision process itself
- [House rules (core)](standards/house-rules-core.md) — the rules for every Craft repository
- [House rules (Rust)](standards/house-rules-rust.md) — extra rules for Rust code
- [Time](standards/time.md) — the calendar exception to the Rust rules

## How a decision is made (the short version)

1. Anyone opens a pull request that adds a file like `rfcs/0004-my-idea.md`. This is called an **RFC** (Request for Comments).
2. A maintainer starts a **comment period** by posting its end date on the pull request: usually **3 days**, or **7 days** for big decisions (adding or removing an app, or changing the process itself).
3. Everyone comments. This is where the real work happens.
4. After the period, the RFC merges when a maintainer (not the author) approves it and no maintainer's objection is left unresolved.

The full text is in [RFC-0000](rfcs/0000-decision-process.md).

## What is being discussed right now

This list is a snapshot from **13 November 2026**. See all open pull requests [here](https://github.com/storytold/craft-contrib/pulls) for the current state.

**How we work together**

| RFC | What it decides |
| --- | --- |
| [#2 · RFC-0001](https://github.com/storytold/craft-contrib/pull/2) | A pull request template so every RFC looks the same |
| [#5 · RFC-0004](https://github.com/storytold/craft-contrib/pull/5) | What happens when two pull requests fix the same bug |
| [#6 · RFC-0005](https://github.com/storytold/craft-contrib/pull/6) | What a good pull request looks like |
| [#8 · RFC-0007](https://github.com/storytold/craft-contrib/pull/8) | What the triage labels mean and how priority is set |
| [#9 · RFC-0008](https://github.com/storytold/craft-contrib/pull/9) | Contributor roles, and how the review workload gets shared |
| [#17 · RFC-0013](https://github.com/storytold/craft-contrib/pull/17) | How an accepted RFC gets changed later |
| [#34 · RFC-0019](https://github.com/storytold/craft-contrib/pull/34) | What belongs in an RFC, and who gets asked to review it |

**Rules and licences**

| RFC | What it decides |
| --- | --- |
| [#33](https://github.com/storytold/craft-contrib/pull/33) | Clarifying contributor credits and local exceptions in the house rules |
| [#41 · RFC-0027](https://github.com/storytold/craft-contrib/pull/41) | Making the licence allowlist match what the apps actually use |
| [#45](https://github.com/storytold/craft-contrib/pull/45) | A new house rule: apps must disclose incomplete functionality |

**Shared code and big features**

| RFC | What it decides |
| --- | --- |
| [#10 · RFC-0009](https://github.com/storytold/craft-contrib/pull/10) | When code may move into the shared library (`craft-libs`) |
| [#15 · RFC-0011](https://github.com/storytold/craft-contrib/pull/15) | Spelling and language data come from one shared place |
| [#12 · RFC-0001 (web)](https://github.com/storytold/craft-contrib/pull/12) | A JavaScript command interface so agents can drive the web builds |
| [#39 · RFC-0023](https://github.com/storytold/craft-contrib/pull/39) | A web runtime so large documents work in a browser tab |
| [#46 · RFC-0030](https://github.com/storytold/craft-contrib/pull/46) | One shared 2D vector drawing layer for three apps |
| [#49 · RFC-0031](https://github.com/storytold/craft-contrib/pull/49) | Hardware video encoding in a separate helper process |
| [#50 · RFC-0032](https://github.com/storytold/craft-contrib/pull/50) | Shared crates for 3D materials and rendering |

**Accessibility, security and CI**

| RFC | What it decides |
| --- | --- |
| [#21 · RFC-0014](https://github.com/storytold/craft-contrib/pull/21) | Accessibility requirements for every UI change |
| [#37 · RFC-0021](https://github.com/storytold/craft-contrib/pull/37) | One e-signature contract for the whole suite |
| [#23 · RFC-0016](https://github.com/storytold/craft-contrib/pull/23) | A hosted connector for using Craft apps from ChatGPT or Claude |
| [#47](https://github.com/storytold/craft-contrib/pull/47) / [#48](https://github.com/storytold/craft-contrib/pull/48) | Locking GitHub Actions and keeping them updated |

## How you can help

**Comment on an open RFC.** This helps the most, and it is the easiest thing to do. A useful comment does not need to be technical:

- "I use PhotoCraft daily. This change would break my workflow because …"
- "I tried something similar in my project. What worked was …"
- "I don't understand this part. Could someone explain it?"

Questions are comments too. If an RFC is hard to understand, saying so is valuable — it means the text needs work.

**Open an RFC.** If you want to change a rule or a decision, add `rfcs/NNNN-short-title.md` in a pull request. Pick the next free number, describe the problem first and the solution second, and say which apps it affects.

**Fix small things.** Typos, broken links and wording fixes are ordinary pull requests and need no RFC.

## Glossary

| Term | Meaning |
| --- | --- |
| **RFC** | A proposal that is discussed on its pull request until a decision is made. |
| **Comment period** | The days when a proposal is open for comments. It is started by a maintainer posting the end date. |
| **Maintainer** | A member of the `craft-app-team` GitHub team. They approve and merge RFCs. |
| **Owner** | An admin of the storytold organisation. They can decide objections. |
| **House rules** | The rules that apply in every Craft repository. They live in `standards/` here and are copied into each repository's `AGENTS.md`. |
| **`AGENTS.md`** | The instruction file that coding agents read in every repository. |
| **`craft-libs`** | The shared Rust library repository that several apps depend on. |
| **Claim** | Saying on an issue that you are working on it, so two people don't fix the same bug ([RFC-0003](rfcs/0003-claiming-issues.md)). |
| **Triager** | Someone with the GitHub Triage role in one app's repository: labelling and closing issues. |
