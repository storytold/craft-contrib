# RFC-0090: Rule 18 covers connected agent entry points and adds a shared conformance check

Updates: [RFC-0072](0072-shared-agents-md.md)

Related: [RFC-0015](0015-confined-file-access.md), [craft-contrib#78](https://github.com/storytold/craft-contrib/issues/78), [craft-libs#2527](https://github.com/storytold/craft-libs/issues/2527)

## Summary

Two changes that close the connected-mode and verification gaps in Rule 18 of `standards/app-agents.md`:

1. Extend Rule 18 so connected and bridged agent entry points (including `--connect`, `--bridge`, `--attach`, and implicit live auto-connect modes) MUST refuse to start unless the target desktop app enforces both `--automation-read-root` and `--automation-write-root` on its control channel.
2. Add a shared CI conformance check in `storytold/craft-common-actions` (`actions/agent-roots-conformance` and `.github/workflows/agent-roots-conformance.yml`) that exercises every Craft app's built `<app>-cli mcp` binary on headless Linux runners.

## Motivation

[RFC-0015](0015-confined-file-access.md) (§1.2 and §6) already requires agent entry points to refuse startup without both automation roots and requires bridge/connected modes to fail closed unless the receiving desktop process enforces the same confinement. However, `standards/app-agents.md` Rule 18 only stated the standalone CLI requirement, and no shared CI check verified it across apps.

An audit of seven Craft applications at `main` on 2026-11-25 shows that only one complies with Rule 18 today, and connected-mode flag names and failure modes have already diverged:

| App (HEAD) | Neither root | One root only | Connected flag / mode | Connected without desktop roots | Status |
| --- | --- | --- | --- | --- | --- |
| WordCraft (`c179a316`) | Refuses (exit 1) | Refuses (exit 1) | `--connect HOST:PORT` | Refuses unconditionally (exit 1) | Compliant |
| PhotoCraft (`e22facf4`) | Starts serving | Starts serving | `--bridge ADDR` / live `AI-MCP` | Starts lazily; connects on first tool call without checking desktop roots | Non-compliant |
| FigCraft (`0ae74129`) | Starts unconfined | Starts serving | `--attach PORT\|HOST:PORT` | Connects without auth or roots check; desktop `--control` can run unconfined | Non-compliant |
| PDFCraft (`6e910675`) | Starts unconfined | Starts serving | None on `mcp` (feature-gated) | N/A | Non-compliant |
| VectorCraft (`99f7de52`) | Auto-connects `127.0.0.1:7979` or starts unconfined | Starts serving | `--connect ADDR` (plus default auto-connect) | Connects with token; no check that desktop enforces roots | Non-compliant |
| FilmCraft (`740adb32`) | Starts unconfined | Unknown option (exit 2) | `--bridge 127.0.0.1:PORT` | Connects with no roots check; root flags unimplemented | Non-compliant |
| EffectCraft (`59aed450`) | Starts unconfined | Starts serving | `--bridge PORT` | Connects unconfined; passing roots with `--bridge` checks CLI args only, leaving desktop footage/services unconfined | Non-compliant |

Three concrete failure patterns emerge from this audit:

- **Silent unconfined fallback or partial startup:** Six of seven audited apps answer MCP `initialize` with zero or one root. Five start completely unconfined when neither root flag is given.
- **Unchecked or lazy connected modes:** PhotoCraft's `--bridge` connects lazily on the first tool call, so it serves MCP `initialize` without ever checking whether the desktop app enforces roots. EffectCraft's `--bridge` accepts root flags on the CLI without confining the desktop process's document/footage reads. FigCraft, VectorCraft, and FilmCraft connect to desktop control servers that can run without roots.
- **Flag divergence and false-pass risk:** Connected modes use `--connect`, `--bridge`, `--attach`, and implicit auto-connect. Meanwhile, FilmCraft exits `2` on `--automation-read-root` because the flag does not exist — a negative-only check that only expects non-zero exit on single-root invocations would misclassify it as passing unless a positive control (`both roots -> answers MCP initialize`) also runs.

## Proposal

### 1. Update Rule 18 in `standards/app-agents.md`

Replace Rule 18 in `standards/app-agents.md` with:

> 18. Agent entry points (`<app>-cli mcp`) MUST refuse to start without `--automation-read-root` and `--automation-write-root` and refuse file access outside them; connected or bridged modes MUST refuse to start unless the target app enforces both roots on its control channel.

This uses RFC 2119 `MUST` throughout (matching RFC-0072 §2 for safety limits) and avoids hardcoding one app's connected-mode flag name.

**Size budget (RFC-0072 §8):** `standards/app-agents.md` grows from 7,993 bytes to 8,107 bytes UTF-8 (+114 bytes), staying under 8 KiB (8,192 bytes). See Open questions below if maintainers interpret "8 KB" strictly as 8,000 decimal bytes.

### 2. Connected and bridged entry-point contract

For any agent entry point that drives a running desktop process (`--connect`, `--bridge`, `--attach`, environment variables such as `*_CONTROL_PORT`, or implicit live auto-connect):

1. **Fail closed before serving MCP:** Before answering MCP `initialize` on stdio, the CLI MUST either refuse connected mode outright (as WordCraft does until its desktop control channel enforces roots) or verify with the target desktop process that both a read root and a write root are enforced on that control channel. Lazy connection on the first tool call does not satisfy Rule 18.
2. **Implicit auto-connect follows the same rule:** An entry point launched without explicit roots MUST NOT silently attach to a default localhost control port or connection file unless that target enforces both roots; otherwise it MUST refuse to start.
3. **Desktop-side enforcement required:** Filtering command arguments on the CLI side while forwarding to an unconfined desktop process does not satisfy Rule 18, because document-referenced assets and desktop services execute inside the desktop process (RFC-0015 §2.3, §6).

### 3. Shared CI conformance check (`storytold/craft-common-actions`)

Add a shared composite action (`actions/agent-roots-conformance`) and reusable workflow (`agent-roots-conformance.yml`) in `storytold/craft-common-actions`. Each Craft app with an `<app>-cli mcp` entry point runs it in CI against its built Linux CLI binary (either from a `gate.yml` artifact or inside the Linux build job).

The runner invokes `<binary> <subcommand>` with ambient `*_AUTOMATION_*_ROOT`, `*_CONTROL_PORT`, and `*_CONTROL_TOKEN` environment variables stripped, sends a live MCP JSON-RPC 2.0 `initialize` request on stdin, and checks:

1. **`no-roots`:** Neither root flag passed -> MUST exit non-zero before answering `initialize`.
2. **`read-root-only`:** Only `--automation-read-root <tmp/read>` -> MUST exit non-zero before answering `initialize`.
3. **`write-root-only`:** Only `--automation-write-root <tmp/write>` -> MUST exit non-zero before answering `initialize`.
4. **`missing-read-root`:** Non-existent read root with valid write root -> MUST exit non-zero before answering `initialize`.
5. **`missing-write-root`:** Valid read root with non-existent write root -> MUST exit non-zero before answering `initialize`.
6. **`both-roots-initialize` (positive control):** Both valid temporary directories -> MUST answer MCP `initialize` with a JSON-RPC 2.0 `result` for `id: 1` (catches CLIs that reject unknown root flags).
7. **`connect-unconfined-refused` (when `connect-flag` is configured):** Binds a local loopback TCP stub (`127.0.0.1:0`) that answers newline-delimited JSON requests with `{"id": <id>, "ok": true, "result": {}}` (advertising no roots), invokes `<binary> <subcommand> <connect-flag> <endpoint> [connect-extra-args]`, and requires the CLI to exit non-zero before answering MCP `initialize`.

**Scope boundary:** This CI check verifies startup confinement and connected fail-closed behaviour deterministically on headless Linux runners without launching a GUI window. Full filesystem escape testing (symlinks, `..`, hard links, UNC/8.3 paths, directory swaps, document-referenced paths) stays in RFC-0015's shared Rust test suite (`craft-libs#2527`).

## Rollout

1. Merge this RFC PR, updating `rfcs/0072-shared-agents-md.md` and `standards/app-agents.md`.
2. Merge the shared conformance action and reusable workflow in `storytold/craft-common-actions`.
3. Propagate the updated `standards/app-agents.md` to Craft repositories through the normal RFC-0072 update flow.
4. Pilot `agent-roots-conformance.yml` in WordCraft first (already compliant).
5. Enable the check in each remaining Craft app as its CLI and control channel comply with Rule 18 (directly or via `craft-libs#2527`), one PR per app.

## Alternatives

- **Rely on RFC-0015 §6 alone without updating Rule 18.** Coding agents read `AGENTS.md` on every task, not every merged RFC; keeping the connected-mode requirement out of Rule 18 lets agents add or modify `--connect`/`--bridge`/`--attach` paths without seeing the fail-closed rule.
- **Require launching each desktop GUI app in the shared CI workflow.** Desktop launches require display/GPU setup and app-specific harnesses. Testing the CLI against an ephemeral loopback stub verifies fail-closed startup in under a second on plain `ubuntu-24.04` runners, while desktop-side enforcement remains tested inside each app's Rust suite.
- **Standardize the connected-mode flag name in this RFC.** Flag naming belongs to command-line conventions rather than Rule 18; taking `connect-flag` and `connect-address-format` as inputs lets every app adopt the check immediately without waiting for flag renames.

## Open questions

1. **8 KB budget unit in RFC-0072 §8:** With the proposed wording, `standards/app-agents.md` is 8,107 bytes UTF-8 (under 8,192 bytes / 8 KiB, over 8,000 decimal bytes). If maintainers treat the limit as strictly 8,000 bytes, should we move connected-mode details into a short linked standard (`standards/automation-roots.md`) and keep Rule 18 under 168 bytes?
2. **Standard control-channel capability probe:** For apps that verify desktop-side roots over a live control channel rather than refusing connected mode outright, should `craft-libs#2527` specify one shared handshake reply field, or is the unconfined loopback stub check sufficient until `craft-libs#2527` lands?
