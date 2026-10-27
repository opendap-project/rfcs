# RFC 0017: Naming and topology: one repository per root, names from the hierarchy

- **Status:** Accepted (2026-10-27). The rules apply to each root at its cut (§7); until a root is cut, its repositories keep today's names.
- **Date:** 2026-10-27
- **Affected repositories:** every public repository: workspace, rfcs, claude-plugins, .github; blackbird-os, system, blackbird-ui, virtual-devices (to the `blackbird` root); t1, board, case, faceplate-touch, faceplate-classic, faceplate-cassette (to the `t1` root); hat-standard, hat-camera, hat-nas, hat-oscilloscope, hat-outdoor, hat-powerbank, hat-radio, hat-recorder, hat-vna (to the `hats` root); kicad-lib, parts-lib, freecad-lib, blender-lib (to the `libs` root); toolkit-host, toolkit-image, toolkit-hardware, toolkit-verify, toolkit-hooks (to the `tools` root); cli, desktop (to the `host` root).
- **Amends:** [RFC 0004](0004-opendap-protocol.md) §11, [RFC 0008](0008-repository-restructure.md), [RFC 0009](0009-odapkit.md), [RFC 0012](0012-composer-and-os-shell.md) (renames only).
- **Related:** [RFC 0018](0018-digital-thread.md) (records and schemas), [RFC 0019](0019-versioning-and-releases.md) (versioning and releases), the [command standard](https://github.com/opendap-project/toolkit-host/blob/main/docs/specs/command-standard.md) (toolkit-host).

## Summary

The public OpenDAP workspace has 33 repositories in GitHub's flat namespace. Their names carry prefixes (`hat-`, `toolkit-`, `-lib`, `odap-`) only to tell them apart there. This RFC groups them into eleven **roots**, one repository each, and names everything inside a root by its position: `<root>/<domain>/<module>` (`hats/standard`, `blackbird/composer`, `tools/electronics`). It sets one identifier rule per kind (crates, binaries, units, runtime paths, reverse DNS, schemas, environment variables, commands, level scales), separates three audiences (maintainers, developers, end users) with a one-way dependency rule, and migrates history-preserving, one root at a time. The `devkit` root is created when its first template exists.

## Motivation

- **Names say where a thing belongs only by convention today.** `odap-*` names device binaries, host tools (until RFC 0009) and the session user; `opendap-*` names crates in the device software, the CLI and the UI alike; `system` and `toolkit` are generic words that will collide with a second device's needs. Inside one repository per domain, the folder is the name and no prefix is needed.
- **Boilerplate drifts across repositories.** Every repository carries its own pre-commit configuration, agent rules, REUSE skeleton and lint settings. Grouping them leaves one copy per root, and doing it while many repositories are still templates moves little history.
- **The audiences are mixed.** Tools that package developers need (cross-building, deploying a development extension, the sysroot, the development key, the emulator) sit in maintainer repositories, behind the maintainers' command. RFC 0009 already forbids `opendap` from needing `odapkit`; the structure should make that boundary visible.
- **Software belongs to the platform it runs on.** The device's binaries, units, runtime paths and bus names are parts of Blackbird OS; naming them after the project (`odap-`, `/run/opendap/`, `org.opendap.*`) hides which layer owns them (the same confusion RFC 0012 fixed for "shell").

## Proposal

### 1. Principles

1. **Names come from the hierarchy.** A thing's name is its position, `<root>/<domain>/<module>`. Prefixes exist only where a flat namespace forces them (GitHub repositories, crates.io, `/usr/bin`, D-Bus).
2. **Each root is one repository.** The hierarchy below a root is folders. Public and private material never share a repository: the boundary stays at repository level, and private repositories (a private overlay, not described here) follow the same rules.
3. **Three audiences, one direction.** Maintainers build the platform; developers build on it with the developers' distribution, the **devkit**; end users use the product, advanced users included. Dependencies flow only maintainers → developers → users: what developers need never requires a maintainer tool, and what users need never requires either.
4. **Data first, prose generated.** Facts that tools or other components read (requirements, blocks, variants, BOM, pinouts, interfaces, device descriptions, tool lists, checks) are data with a schema; their Markdown is generated and checked.
5. **One engine, many surfaces.** Logic lives in a tested library; commands, hooks, checks and devkit plugins call it.

### 2. Roots

| Root (repository) | From today | Contents |
| --- | --- | --- |
| `workspace` | workspace | Manifest, contributor guide, templates rendered into every root, project assets |
| `blackbird` | blackbird-os, system, blackbird-ui, virtual-devices, `odap-inject` (toolkit-verify) | Everything that runs on the device platform: image, composer, OS Shell, services, UI, SDK, first-party apps, the emulator, device-side test instrumentation |
| `t1` | t1, board, case, faceplate-touch, faceplate-classic, faceplate-cassette | The T1: product, board, case, faceplates and the interfaces it owns |
| `hats` | hat-standard, hat-camera, hat-nas, hat-oscilloscope, hat-outdoor, hat-powerbank, hat-radio, hat-recorder, hat-vna | The HAT standard, conformance, reference designs; HATs without design work as concepts |
| `libs` | kicad-lib, parts-lib, freecad-lib, blender-lib | Design libraries (data) shared by maintainers and developers |
| `tools` | toolkit-host, toolkit-image, toolkit-hardware, toolkit-verify, toolkit-hooks, the workspace's `scripts/` | The maintainers' tools and their command |
| `host` | cli, desktop | What runs on the user's computer: the `opendap` CLI and OpenDAP Desktop, with their shared crates |
| `devkit` | new | The developers' entry point: release manifest, templates, tested samples, developer portal, compatibility matrix |
| `rfcs` | rfcs | Unchanged |
| `claude-plugins` | claude-plugins | Unchanged (the contributor plugin) |
| `.github` | .github | Unchanged |

Inside a root, every hardware or product component follows one archetype (`component.toml`, `docs/`, `interfaces/`, `blocks/`, `variants/`, `hardware/<pcb>/`, `studies/<domain>/<study>/`, `verification/`, `export/`, `check.toml`; folders exist only with content). The archetype and each root's layout are specified in the workspace's contributor guide, each root's layout at its cut (§7).

### 3. Identifiers

| Kind | Rule | Examples (today → new) |
| --- | --- | --- |
| Repository | The root's name | `hat-standard` → `hats` (folder `standard/`) |
| Rust crate | `<root>-<path joined by ->` | `odap-composer` → `blackbird-composer`; `opendap-input` → `blackbird-ui-input`; `opendap-core` (CLI) → `opendap-host-core` |
| Internal daemon | `/usr/lib/blackbird/<module>`, no prefix | `/usr/bin/odap-composer` → `/usr/lib/blackbird/composer`; `odap-product` → `/usr/lib/blackbird/player` |
| systemd unit | `blackbird-<module>.<type>` | `odap-composer.service` → `blackbird-composer.service`; `odap-power.socket` → `blackbird-power.socket` |
| PAM service, udev rules | `blackbird-<module>` | PAM `odap-composer` → `blackbird-composer`; `70-opendap-boot.rules` → `70-blackbird-boot.rules` |
| Runtime and state paths | `/run/blackbird/<module>/`, `/var/lib/blackbird`, `/usr/share/blackbird`; session user and group `blackbird` | `/run/opendap/composer` → `/run/blackbird/composer`; `/run/opendap/power.sock` → `/run/blackbird/power/control.sock`; debug flags under `/run/blackbird/debug/`; user `odap` → `blackbird` |
| Bus and reverse-DNS names | `cc.opendap.Blackbird.<Module>` (the project's domain is `opendap.cc`) | `org.opendap.hw` → `cc.opendap.Blackbird.Hw`; `org.opendap.system.Power` → `cc.opendap.Blackbird.Power` |
| Schema | Cross-domain contracts `opendap.<kind>/vN`; OS-internal `blackbird.<kind>/vN` | `opendap.vdev-status/v1` → `blackbird.emulator-status/v1`; `opendap.component/v1` unchanged |
| Environment variable | By audience: `OPENDAP_*` for the `opendap` CLI and its plugins; `ODAP_*` for the maintainers' command and tools (`ODAP_<ROOT>_DIR`); `BLACKBIRD_*` for the OS at run time | `ODAP_IMAGE` → `OPENDAP_IMAGE`; `ODAP_VIRTUAL_DEVICES_DIR` → `ODAP_BLACKBIRD_DIR`; `OPENDAP_TOOLKIT_HOOKS` → removed |
| Command | Developers and users: `opendap` (aliases `odap`, `dap`), plugins `opendap-<tool>`. Maintainers: `odw`, plugins `odw-<tool>`, grammar `odw <tool> <verb>` with tool names short and unique, no discipline prefix | `odapkit image build` → `odw image build`; `odapkit-deploy` → `opendap deploy`; `odap-vdev` → `opendap emulator` |
| KiCad project | Folder by position; project name and board identity unchanged | `board/hardware/` → `t1/board/hardware/main-board/`; project and board identity stay `t1-main-board`, so sheet UUIDs do not change |
| ADR series | One series per component, in its `docs/adr/`, cited as `<component> ADR NNNN` | `board ADR 0013`, `blackbird/os ADR 0009`; numbers never change in a merge |
| Requirement IDs | Never renumbered; new prefixes are allocated from one workspace list | `TK` stays for the CLI; new tool requirements get their own prefix |
| Level scales | One letter per scale ([RFC 0018](0018-digital-thread.md) §7): Q quality ([RFC 0010](0010-quality-levels.md)), I integration, V software verification, T test level, C HAT compatibility, H hardware support | integration `L0–L6` → `I0–I6`; software verification `L0–L5` → `V0–V5`; test levels `L0–L4` → `T0–T4` |
| Contributor plugin commands | `/opendap:<domain>-<task>` | `/opendap:requirement` → `/opendap:docs-requirement`; `/opendap:fab-package` → `/opendap:electronics-fab-package` |

`opendap` and `odap` remain only for what is project-wide: the brand, the `opendap` CLI, the OpenDAP Protocol's DNS-SD service type and URI scheme, cross-domain schemas, host paths under `~/.local/share/opendap/` and `~/.config/opendap/`, and the maintainers' environment variables.

### 4. Audiences and the dependency rule

| Audience | Command | Root(s) | May depend on | Never depends on |
| --- | --- | --- | --- | --- |
| Maintainers | `odw` | `tools`, and each root's maintainer tools | everything below; `opendap` for device operations | — |
| Developers | `opendap` and its plugins; the devkit | `devkit`, `host`, developer plugins shipped by `blackbird` and `tools` | published outputs of every root (exports, schemas, images, SDK) | `odw`, maintainer tools, a workspace checkout |
| End users | the product, OpenDAP Desktop, `opendap` | `host`, released images | released artifacts | developer or maintainer tools |

Every tool declares its audience and stability in its root's `tools.toml`; `<command> help` lists both. A check verifies the rule: no developer or user tool calls, requires the path of, or hints only at a maintainer tool.

### 5. Tool graduation

A maintainer tool that developers need **graduates**: it becomes an `opendap-<tool>` plugin with `audience = "developer"`, its old name is kept as a deprecated alias for one minor release (`replaces` in `tools.toml`), and it stops reading maintainer-only paths or variables. The first graduates are `cross`, `deploy`, `dev-key`, the sysroot's install step, and the emulator (`odap-vdev` → `opendap emulator`). Building the sysroot and building images stay maintainer tools. Tool names, `tools.toml`, help, `--json`, exit codes and the conformance test are specified in the [command standard](https://github.com/opendap-project/toolkit-host/blob/main/docs/specs/command-standard.md), accepted with this RFC (toolkit-host today, `tools/workspace` after the `tools` cut).

### 6. What this amends

| RFC | Text | Amendment |
| --- | --- | --- |
| 0004 §11 | Technical identifiers use the short form `odap` for the DNS-SD service type, a URI scheme, crate and package names, and interface namespaces where `org.opendap.*` does not apply; the Summary names the device's `org.opendap.*` interfaces | The DNS-SD service type (`_odap._tcp`) and URI scheme keep `odap`. Crates and packages follow §3 (`blackbird-bus-*`, `opendap-host-*`). Device interfaces are `cc.opendap.Blackbird.<Module>` (`org.opendap.media.Player` → `cc.opendap.Blackbird.Media.Player`); the protocol's own namespaces are an open question (below). |
| 0008, Repositories | One repository per audience and domain, T1 repositories under `devices/t1/`; "renaming `board`, `case` and the faceplates on GitHub" rejected | Replaced by the roots of §2. Grouping by folder happens inside the `t1` root instead of in the workspace layout. |
| 0008, Interfaces → Finding repositories | `repos.tsv` or `ODAP_<REPOSITORY>_DIR` | `repos.tsv` or `ODAP_<ROOT>_DIR`; tools resolve paths inside a root by its layout, never by a repository-name pattern. |
| 0008, Interfaces → Names | Device binaries and units `odap-*`; session user `odap` with home `/var/lib/odap`; `ODAP_*`; reverse DNS, `/run/opendap/` and `opendap.<kind>/vN` keep the full name; crates keep `opendap-*` until the SDK is published | Replaced by §3. |
| 0008, Requirement IDs | Never renumbered; TK stays in `cli` | Unchanged; TK moves with the CLI into `host`. |
| 0009, The command | `odapkit TOOL` dispatches `odapkit-TOOL`, found in the `bin/` folders of repositories matching `toolkit-` in `repos.tsv`, then on `PATH`; `odapkit help` shows each tool's first help line | The maintainers' command is `odw`, plugins `odw-<tool>`. Discovery reads each root's `tools.toml`, then `PATH`; `help` lists summary, audience and stability. The dispatcher lives in `tools/workspace`. |
| 0009, Names | `odapkit-*` host tools; device binaries, crates and `ODAP_*` stay `odap`; `odap-vdev` keeps its name | `odapkit-*` → `odw <tool>` or, when graduated, `opendap <tool>` (§5); device names follow §3; the emulator is `opendap emulator`, shipped by `blackbird`. |
| 0009, The boundary | `opendap` never needs `odapkit`; `odapkit` may use `opendap`; the emulator never uses `odapkit` | Kept, generalised to the three audiences (§4). `ODAP_IMAGE` becomes `OPENDAP_IMAGE`. |
| 0012, Renames | `odap-composer` crate and binary, `odap-composer.service`, PAM `odap-composer`, `/run/opendap/composer/` | `blackbird-composer` crate, `/usr/lib/blackbird/composer`, `blackbird-composer.service`, PAM `blackbird-composer`, `/run/blackbird/composer/`. The composer/OS Shell vocabulary is unchanged. |

For each root, RFC 0008's and RFC 0009's rules stay in force until that root's cut (§7); nothing public is renamed or moved outside a cut.

### 7. Migration

One root at a time, lowest risk first: **`libs` → `tools` → `hats` → `host` → `t1` → `blackbird`**. `devkit` is created when its first template exists. Each cut:

1. **Freeze** the source repositories (announced; open work is pushed first).
2. **Move with history**: each source repository's history is rewritten into its folder of the root (`git filter-repo --to-subdirectory-filter`, then merged), so `git log --follow` and blame keep working. The cut records each source repository's last commit.
3. **Apply the rename map**: every move, rename, split, merge and conversion is one row of a data file (`old`, `new`, `kind`, `action`), applied by a tool, never by hand. Annex A summarises it.
4. **Rewrite links**: links between repositories are GitHub URLs; the tool rewrites every URL and path the map touches, and a link check lists what is left. GitHub's redirects cover renamed repositories; merged ones are covered by step 6.
5. **Render the shared configuration once** per root from the workspace templates, and run the root's checks (pre-commit, ERC/DRC parity for boards, builds and tests for software).
6. **Archive** each source repository on GitHub with a README that points to its new folder.
7. **Update** `repos.tsv`, the bootstrap script and the contributor guide.

Device-side renames (§3) land with the `blackbird` cut and one image rebuild; host-side state under `~/.local/state/opendap/` is moved by the tool that owns it on first run.

## Alternatives

- **Keep the flat repositories and only rename (`t1-board`, `blackbird-composer` as repositories).** Not chosen: GitHub's flat namespace keeps the prefixes, every repository keeps its own boilerplate, and the hierarchy exists only in the workspace layout. RFC 0008 rejected renames to avoid breaking links; the migration above rewrites them by tool.
- **One monorepo for everything public.** Not chosen: roots have different audiences, licences (CERN-OHL-S/-W, GPL, MPL, CC-BY-SA), release cadences and reviewers; one repository per root keeps those boundaries.
- **Git submodules or subtrees instead of history-preserving merges.** Not chosen: submodules keep the flat repositories and their drift; subtree merges without path rewriting break `git log --follow`.
- **Keep `odap-` for device binaries (RFC 0008/0009).** Not chosen: the binaries belong to Blackbird OS, and a derivative or a second device runs the same platform.
- **`org.opendap.*` reverse DNS.** Not chosen: reverse DNS follows a domain the project controls, `opendap.cc`.
- **Rename `odapkit` to another `opendap`-like word.** Not chosen: the maintainers asked that the maintainers' command be told apart from `opendap` and its aliases (RFC 0009); `odw` is short, distinct and names the workspace.

## Decisions

Accepted on 2026-10-27, each with its basis:

- **Topology: one repository per root (§2).** Basis: the flat namespace forces the prefixes (Motivation); one copy of the shared configuration per root instead of one per repository; the hierarchy can only be folders inside a repository. The cut-over runs per root, lowest risk first, with history preserved (§7).
- **Identifiers: §3 as a whole**, superseding RFC 0008's names and RFC 0004 §11 as §6 states. Basis: names come from the hierarchy (§1 rule 1); the device's binaries, units, runtime paths and bus names belong to Blackbird OS, as RFC 0012 already applied to the composer.
- **Blackbird's scope.** Blackbird is the whole device platform: image, composer, OS Shell, services, UI, SDK, first-party apps named by function (`player`) and the emulator (`blackbird/emulator`); "system" stops being a separate entity. Basis: everything in that list runs on, or emulates, the device platform (Motivation, last item).
- **The `devkit` root.** `devkit` is its own root: release manifest, templates, tested samples, the developer portal and the compatibility matrix, which belong to no other domain. Basis: the audience rule (§4) and RFC 0009's boundary (`opendap` never needs the maintainers' command).
- **The maintainers' command is `odw`.** Basis: RFC 0009 requires it to be told apart from `opendap` and its aliases `odap` and `dap` (TK-003); `odw` is short, distinct and names the workspace.
- **VD-004 is reworded**: the emulator is its own module with its own requirements inside the `blackbird` root (`blackbird/emulator`), not its own repository. Basis: the Blackbird scope above; VD-004's intent, keeping the emulator apart from the toolkit and the operating system, is kept as a module boundary.
- **Template repositories become concepts.** The seven HAT repositories scaffolded from the template (hat-camera, hat-nas, hat-oscilloscope, hat-powerbank, hat-radio, hat-recorder, hat-vna) fold into `hats/concepts/<hat>/`, and faceplate-classic and faceplate-cassette into `t1/faceplates/<name>/` as concepts; their GitHub repositories are archived with a pointer. A HAT or faceplate gets design folders when its design starts. Basis: on 2026-10-27 their `hardware/`, `case/`, `firmware/` and `software/` folders hold only placeholder READMEs.
- **The HAT standard is named "OpenDAP HAT Standard"**: the interface between the T1 and its HATs, both sides; folder `hats/standard/`. Basis: from interface v2 it is an electrical USB interface, not a case ([hat-standard ADR 0006](https://github.com/opendap-project/hat-standard/blob/main/docs/adr/0006-hats-are-usb-only.md), [RFC 0014](0014-hat-interface-v2-usb-only.md)).
- **Hardware roots consume `libs` through a pinned tag resolved by the workspace manifest**: one copy in the workspace, the version recorded in each component's `consumes`; no git submodules. Basis: the component export contract (consumers read only a tagged export, never a producer's sources); a submodule per consumer would keep one copy per repository.

## Open questions

- **Protocol namespaces.** RFC 0004 §11 allows interface namespaces outside the device's. Under §3 they would be `cc.opendap.<Name>`; the exact identifiers stay in the protocol specification.

## Annex A. Rename map summary

The rename map lists every public path and identifier the migration changes, one row each, with its kind and action. Counts on 2026-10-26: **2,369 rows**.

### By action

| Action | Rows | Meaning |
| --- | ---: | --- |
| move | 717 | Same content, new path |
| rename | 385 | Identifier or file renamed (crates, binaries, units, paths, bus names, commands) |
| merge | 360 | Several sources become one (mostly per-repository configuration rendered once per root) |
| delete | 225 | Removed: template placeholders, duplicates, superseded files |
| keep | 224 | Listed because a rule applies to it, but unchanged |
| convert | 212 | Prose to data with a schema (generated Markdown), or format change |
| split | 106 | One source becomes several (large specs, generators, tools by audience) |
| generate | 70 | Becomes a generated file with a `--check` |
| extract | 62 | Data or a library extracted from a larger file |
| archive | 8 | Kept read-only with a pointer |

### By kind

| Kind | Rows | Actions |
| --- | ---: | --- |
| config | 463 | merge 308, delete 41, keep 34, move 29, rename 29, convert 8, split 8, generate 6 |
| data | 339 | convert 178, move 88, extract 51, keep 8, rename 5, split 5, generate 4 |
| file | 215 | move 147, delete 40, split 23, merge 3, convert 1, generate 1 |
| template | 145 | delete 105, move 21, generate 18, keep 1 |
| doc | 137 | keep 44, move 37, split 23, convert 13, delete 10, merge 5, generate 4, rename 1 |
| generated | 99 | move 67, generate 27, merge 2, keep 1, delete 1, rename 1 |
| fixture | 87 | move 59, keep 27, generate 1 |
| identifier | 84 | rename 62, keep 18, merge 2, convert 1, delete 1 |
| library (manufacturer keys) | 72 | rename 64, merge 4, move 3, keep 1 |
| adr | 70 | keep 39, move 31 |
| asset | 69 | move 42, delete 20, keep 6, generate 1 |
| schematic | 61 | move 59, rename 2 |
| study | 58 | move 45, archive 8, split 4, convert 1 |
| test | 48 | move 36, keep 5, split 4, generate 2, merge 1 |
| env | 46 | keep 23, rename 15, delete 6, merge 2 |
| command | 42 | rename 31, convert 6, split 2, merge 1, delete 1, move 1 |
| binary | 39 | rename 38, keep 1 |
| crate | 38 | rename 25, extract 8, split 3, merge 1, move 1 |
| rdns | 38 | rename 38 |
| runtime-path | 37 | rename 35, keep 2 |
| generator | 36 | split 21, move 13, merge 2 |
| repo | 31 | merge 28, split 2, keep 1 |
| schema | 26 | rename 11, keep 10, generate 2, extract 2, move 1 |
| engine | 22 | split 10, move 9, convert 3 |
| unit | 19 | rename 18, keep 1 |
| tool | 16 | move 14, extract 1, split 1 |
| font | 10 | move 10 |
| contributor plugin command (`/opendap:…`) | 10 | rename 10 |
| hook | 6 | move 3, keep 2, merge 1 |
| dir | 3 | generate 3 |
| pcb | 2 | move 1, generate 1 |
| submodule | 1 | convert 1 |

### Repositories

| Today | New |
| --- | --- |
| workspace | `workspace` (kept) |
| blackbird-os, system, virtual-devices | `blackbird` (`os/`; contexts by layer; `emulator/`) |
| blackbird-ui | `blackbird` (`ui/`); project brand assets to `workspace` (`assets/brand/`) |
| t1, board, case, faceplate-touch, faceplate-classic, faceplate-cassette | `t1` (`product/`, `board/`, `case/`, `faceplates/<name>/`) |
| hat-standard | `hats` (`standard/`) |
| hat-camera, hat-nas, hat-oscilloscope, hat-outdoor, hat-powerbank, hat-radio, hat-recorder, hat-vna | `hats` (`concepts/<hat>/`) |
| kicad-lib, parts-lib, freecad-lib, blender-lib | `libs` (`kicad/`, `parts/`, `freecad/`, `blender/`) |
| toolkit-host, toolkit-hooks, toolkit-hardware, toolkit-verify | `tools` (`workspace/`, `core/`, `env/`, `electronics/`, `parts/`, `sim/`, `mechanical/`, `measure/`, `schemas/`); `odap-inject` to `blackbird` (`testing/inject/`) |
| toolkit-image | `tools` (`image/`, `env/`); developer tools to `host` as `opendap` plugins |
| cli, desktop | `host` (`cli/`, `desktop/`, `crates/`) |
| rfcs, claude-plugins, .github | unchanged |
