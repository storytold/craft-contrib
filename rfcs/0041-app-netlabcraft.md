# RFC-0041: Add NetLabCraft to the suite

Builds on [RFC-0000](https://github.com/storytold/craft-contrib/blob/main/rfcs/0000-decision-process.md),
under which adding an app is an RFC with a 7-day comment period. The sections follow the outline of
the proposal in craft-contrib PR #3 (closed by its author, never merged). File to add: `rfcs/0041-app-netlabcraft.md`.

## Name and id

Display name **NetLabCraft**, machine id `netlabcraft` (binaries `netlabcraft` and `netlabcraft-cli`,
packages `netlabcraft-<version>-<platform>-<arch>.<ext>`). Today the app id is
`io.github.ayozetr.netlabcraft`; if the app joins the suite it would move to `ai.storyteller.netlabcraft`.
(Open question below.)

## What it is

A network simulator for learning and teaching: build a network, configure routers and switches with
the console commands taught in networking courses, and watch every Ethernet frame cross it (captures
export as valid pcap). Desktop app (egui), CLI, MCP server and a browser (wasm) build, from one Rust
codebase. For students, teachers and lab authors. The familiar app it is like: Cisco Packet Tracer,
named only to say what NetLabCraft is like (see RFC-0029).

Repository: https://github.com/ayozetr/netlabcraft (MIT OR Apache-2.0). Six releases so far
(v0.1.0 on 2026-11-13, v0.6.0 on 2026-11-16) with installers for Linux (x86_64 and aarch64), Windows,
macOS, FreeBSD and the web (not code-signed yet).

## Fit

It is its own app: nothing in the suite simulates networks. It overlaps no existing app. It already
uses the suite's egui fork, the layered-crate structure, the never-crash and safe-Rust rules, the
"everything is a command" model, and an MCP server. The community request thread in `#app-requests`
("CraftTracer - Cisco PacketTracer alternative", 20 votes) asks for exactly this app.

## First release and where it is today

The first release (v0.1.0) was published on 2026-11-13; the latest is v0.6.0 (2026-11-16). Numbers for the
current main branch (a few features, such as Dial-up, are newer than v0.6.0), counted from the repository:

- 170 engine commands, 1341 console grammar entries, 65 device models.
- 48 numbered labs with a walkthrough each, checked by the test suite, plus sample labs.
- Switching and routing: VLANs, STP/RSTP, EtherChannel, OSPF (areas, stub, NSSA, virtual links), EIGRP,
  RIP, BGP-4 (sessions, route reflectors, confederations, policy), route maps, prefix lists,
  redistribution, ACLs, NAT, DHCP, DNS, IPv6, IPsec and remote-access VPN, PPPoE and Dial-up.
- Wi-Fi with WEP, WPA and WPA2, IoT things and programmable boards, a cellular network, a sniffer with
  port mirroring, NetFlow, Simulation mode with pcap export, a physical workspace.
- Six interface languages (Spanish, Portuguese (Brazil), French, Italian, German, Simplified Chinese);
  none has been reviewed by a native speaker yet.
- Agent control over MCP and the CLI; every feature is a command with tests.

Not included: code-signed packages, importing the reference app's lab files (out of scope on legal
grounds, see `docs/decisions.md`), WPA3 (the reference app does not have it). What is still missing is
ranked in `docs/gaps.md`.

## Shared code

It does not use `craft-libs` crates today. Candidates once it joins: the GPU layer (RFC-0022) and the
suite's i18n data, instead of its own catalogs (`locales/<locale>/netlabcraft.tsv`).

## Clean room

The reference app was only ever observed from the outside, with synthetic labs. Nothing inside its
installation was read, disassembled or copied; no vendor icons, artwork, help text, sample labs or
output wording; no GPL/LGPL/AGPL code was read or copied. Every asset has a row in `ATTRIBUTION.md`
(`cargo xtask assets` enforces it). Cisco and Packet Tracer are used only to describe what the app is
like, and `NOTICE` says it is not affiliated with Cisco.

## Maintainers

@ayozetr (author and sole maintainer so far).

## Prototype

The repository above, already released.

## Open questions

- Does the app move to `storytold/netlabcraft` (and `ai.storyteller.netlabcraft`), or stay at its
  current home and join the launcher catalog from there?
- Releases: the current workflow is self-contained and builds unsigned packages for six targets. Should it move to
  `craft-common-actions`, and if so with which owner-side setup (signing)?
- `docs/gaps.md` and the parity pages are one shared table each; RFC-0018 (items as files) would fit
  them. Does the app adopt it once accepted?
