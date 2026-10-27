# RFC 0008: Repositories by audience and domain

- **Status:** Accepted (2026-10-23).
- **Amended by:** [RFC 0009](0009-odapkit.md) (the maintainers' tools are `odapkit-<tool>`), [RFC 0012](0012-composer-and-os-shell.md) (`odap-shell` is `odap-composer`), [RFC 0017](0017-naming-and-topology.md) (one repository per root and names from the hierarchy, applied per root at its cut)
- **Date:** 2026-10-23
- **Affected repositories:** workspace, cli (from toolkit-cli), toolkit-image (new), toolkit-host, toolkit-verify, virtual-devices, blackbird-os, system, blackbird-ui, t1, board, case, faceplate-touch, faceplate-classic, faceplate-cassette (the faceplates), claude-plugins, .github.

## Summary

Each repository now belongs to one audience and one domain, and depends only on others' documented outputs.

- **The `opendap` CLI** leaves the toolkit for its own repository, `cli`, next to Desktop. It is OpenDAP's adb: for advanced users and package developers who use the SDK and the emulator.
- **The Developer Toolkit** is the platform maintainers' set of tools, in several repositories and several forms. The new `toolkit-image` repository takes the image, sysroot, development key, cross-build and deploy tools out of blackbird-os and system.
- **Technical identifiers use `odap`:** binaries, units, host tools and environment variables.
- **The T1's own repositories** are grouped under `public/devices/t1/`. Generic ones (OS, system software, UI, virtual devices, CLI, Desktop, toolkit, HATs) stay outside it.

## Motivation

- **The CLI is user-side:** the `opendap` CLI is for advanced users and package developers who use the SDK and the emulator, not for the maintainers who develop them, like adb.
- **The toolkit is not a CLI:** it covers every tool the maintainers need, in several repositories; some are CLIs, others scripts.
- **Virtual devices are their own repository** (RFC 0007).
- **The structure follows the domain:** repositories follow the domain model, its concerns and who uses what.
- **Tools are real toolkit tools:** modular, one task each, with good defaults and domain naming. Technical names use `odap`.

## Proposal

### Repositories

| Workspace path | Repository | Audience | Contents |
|---|---|---|---|
| `public/cli` | `cli` (renamed from `toolkit-cli`) | Advanced users, package developers | The `opendap` binary: devices, access, capabilities, OS updates, official images, chips and HATs, the emulator as a target, discovery, MCP. The host crates stay here until Desktop consumes them; a `client` repository is created only when a second consumer exists (Rust crate architecture §6) |
| `public/desktop` | `desktop` | Users | Unchanged |
| `public/toolkit/image` | `toolkit-image` (new) | Maintainers | `odap-image` (mkosi in the build VM), `odap-sysroot`, `odap-dev-key`, `odap-security-check`, `odap-cross`, `odap-deploy`, the build VM's definition |
| `public/toolkit/host`, `hooks`, `hardware`, `verify` | unchanged names | Maintainers | toolkit-verify takes `odap-inject` (device-side test instrumentation) |
| `public/virtual-devices` | `virtual-devices` | Package developers and maintainers | `odap-vdev`: one tool with domain subcommands, a JSON status for other tools; the composer; layers, models, the virtual BSP |
| `public/blackbird-os` | `blackbird-os` | Platform | The image definition only. Its build inputs (development key tree, board support packages) are passed by `odap-image`; it refers to no other repository |
| `public/system` | `system` | Platform | Device software and SDK; no host scripts |
| `public/blackbird-ui` | `blackbird-ui` | Platform | Fonts published in `export/fonts/` like the tokens; the Blackbird OS mockups from t1 |
| `public/devices/t1/t1`, `…/board`, `…/case`, `…/faceplates/*` | unchanged names | The T1 | Only T1 content. The generic specs and UI mockups move to system and blackbird-ui |
| `public/hats/*`, `public/libs/*`, `public/rfcs`, `public/claude-plugins`, `public/.github` | unchanged | — | Paths and links updated |

### Interfaces

- **Finding repositories.** A tool finds another repository through the workspace's `repos.tsv`, by repository name, or through `ODAP_<REPOSITORY>_DIR` (for example `ODAP_BLACKBIRD_OS_DIR`). Guessed relative paths are not used.
- **Outputs.**
  - blackbird-os: `image/mkosi.output/`.
  - toolkit-image: the development key at `~/.local/share/opendap/dev-ssh/` and the sysroot volume at `~/.cache/opendap/sysroot/`.
  - virtual-devices: `odap-vdev status --json`.
  - blackbird-ui: `export/` and `brand/svg/`.
- **Names.**
  - Device binaries and units: `odap-shell`, `odap-product`, `odap-boot`, `odap-inject`.
  - The session user is `odap`, with home `/var/lib/odap`.
  - Environment variables: `ODAP_*`.
  - Reverse-DNS identifiers, paths under `/run/opendap/` and schema names (`opendap.<kind>/vN`) keep the full name.
  - Crate names keep `opendap-*` until the SDK is published, where they become a versioned decision.
- **Links.** Links between repositories are GitHub URLs, never relative paths.

### Requirement IDs

- IDs are never renumbered.
- The CLI's requirements keep their TK IDs in `cli`.
- The CLI's duplicate `TK-012` becomes `TK-018`.

## Alternatives

- **A `client` repository for the host crates now.** Not chosen: Desktop has no code yet, and the crate architecture moves crates when a consumer appears.
- **Renaming `board`, `case` and the faceplates on GitHub (`t1-board`…).** Not chosen: grouping them under `devices/t1/` gives the same clarity without breaking the 70 or so links to them.
- **Keeping the toolkit as one CLI with plugins.** Not chosen: the toolkit is a set of tools, and the CLI is user-side.

## Open questions

- Restating at SYS level the T1 requirements that describe generic software behaviour (T1-045 to T1-047, T1-051 to T1-053, T1-056, T1-061 to T1-068): their IDs are confirmed and change only by the maintainers' decision.
- Whether the CLI lists the toolkit's tools (`opendap tools`).

## Inventory

The analysis behind this RFC (2026-10-23) lists each repository's purpose, contents, consumers and misplacements, and every path and link a move breaks. Its main findings are summarised in Motivation.
