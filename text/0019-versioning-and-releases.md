# RFC 0019: Versioning and releases

- **Status:** Accepted (2026-10-27). Nothing is tagged yet; the repository changes in §11 are listed, not applied.
- **Date:** 2026-10-27
- **Affected repositories:** every repository that tags or releases; first toolkit-host, toolkit-hardware, toolkit-hooks, board, t1, faceplate-touch, case, hat-standard, system, blackbird-os, cli, desktop, virtual-devices, rfcs. The names are today's; [RFC 0017](0017-naming-and-topology.md) groups them into one repository per root, which this RFC anticipates (§2).

## Summary

OpenDAP versions six kinds of things, each with its own scheme: software and component releases use Semantic Versioning in tags prefixed by their path (`<path>/vX.Y.Z`); hardware revisions are `<board>-rN` and always carry a quality level (RFC 0010); interfaces are `<iface>/vN.M`; a component export carries the tag it was generated at; schemas are `opendap.<kind>/vN`; Blackbird OS images are numbered versions (Blackbird OS ADR 0011). Every release writes a manifest, `opendap.release/v1`, that pins what it contains (component tags, the OS version, interface and schema versions), the quality level each variant reached and the debt left above it. Changelogs are generated. Developer surfaces get a deprecation policy; internal maintainer tools do not. A hardware release is verified per variant, never by inheritance (RFC 0011 rule 9).

## Motivation

- **Nothing is tagged yet.** No public repository has a tag today, while the component export contract requires every export to be pinned to a tagged version of its producer (rules 2 and 4) and leaves the tag format open ("Formats of `name`, `version` and interface identifiers").
- **Requirements already ask for versions.** Images are signed, versioned and published as releases (OS-011); Blackbird OS releases are identified by number only (Blackbird OS ADR 0011); user-facing versions are numbers following SemVer (system user-facing conventions); each CLI binary follows SemVer and never breaks a published JSON schema or error code (CLI spec A.11).
- **Quality levels need a place to be recorded.** Each build declares its target level in its release notes and items above it are visible debt (RFC 0010 rules 2 and 3); a level once reached is kept (rule 4). Each variant carries its target level and is verified on its own (RFC 0011 rule 9). Without a manifest, a release cannot state either.
- **Interfaces are versioned informally.** The faceplate interface went from v0 to v1 (RFCs 0001, 0013); the HAT interface from v1.1 to v2, "a new major revision: v1 and v2 do not mate" (RFC 0014). The rule behind major and minor numbers is not written down.
- **One repository per root breaks plain tags.** When several components share a repository, `v1.0.0` no longer says which component it releases.

## Proposal

### 1. What is versioned

| Kind | Scheme | Example | Where it is written |
|---|---|---|---|
| Software or component release | SemVer 2.0.0, tag `<path>/vX.Y.Z` | `composer/v0.3.0`, `v0.3.0` in a single-component repository | Git tag; `component.toml` or the package manifest (`Cargo.toml`, `pyproject.toml`) |
| Hardware revision | `<board>-rN`, with its quality level | `t1-main-board-r1` (target Q1) | Git tag; the release manifest |
| Interface | `<iface>/vN.M` | `hat/v2.0`, `faceplate/v1.0` | The interface record (`opendap.interface/v1`); git tag in the owning repository |
| Component export | The producing component's tag | `version: t1-main-board-r1` | `export/component.yaml` |
| Schema | `opendap.<kind>/vN`, `blackbird.<kind>/vN` | `opendap.board-variant/v1` | Each record's `schema` key |
| Blackbird OS image | SemVer number, no codename | `Blackbird OS 1.0.0` | `/etc/os-release`, the image file name, the manifest |

### 2. Software and component releases

1. **Semantic Versioning 2.0.0.** MAJOR for an incompatible change of the component's public surface, MINOR for a compatible addition, PATCH for a compatible fix. The public surface is what others consume: a crate's public API, a command's arguments, exit codes and `--json` output, a published schema, an exported file, an implemented interface.
2. **Path-prefixed tags.** In a repository with several releasable components, a tag is `<path>/vX.Y.Z`, where `<path>` is the component's folder relative to the repository root (`ui/components/v0.2.0`). A repository that releases as one unit uses `vX.Y.Z`. This is the convention Go modules use for nested modules; it keeps each component's history ordered by `git tag --list '<path>/v*'`.
3. **Before 1.0.** A MINOR may break the human-readable surface, never a published schema version or a documented error code (CLI spec A.11, applied to every component).
4. **Pre-releases** use SemVer identifiers (`v1.0.0-rc.1`). Development builds carry build metadata with the commit (`v1.0.0-dev+3f2a1c9`) and are never tagged.
5. **Components with exports.** Removing or renaming an exported file, or moving an implemented interface to a new major, is a MAJOR release of the component; adding a file or an interface is a MINOR.
6. **Annotated tags only**, created from a clean tree at a commit whose `release` stage passes (§8).

### 3. Hardware revisions

1. **A revision is what was ordered.** `<board>-rN` (`N` from 1) is tagged on the commit whose fab package is sent to manufacture; the board identity (`t1-main-board`) is already unique in the project. Iterations that are not ordered are not tagged.
2. **Sub-revisions.** A change to the PCB is a new revision `rN`. A change to the BOM or the population only, on the same PCB, is a sub-revision `rN.M` (`t1-main-board-r1.1`, `M` from 1), tagged and released like a revision. Population options (BRD-053) are variants (RFC 0011), not revisions.
3. **A revision covers every variant** of the board: the variants are data (RFC 0011 rule 7) and are listed with their own target and reached levels in the revision's manifest.
4. **Every revision carries a quality level.** The manifest states the target level when the revision is tagged (RFC 0010 rule 2) and the level each variant reached once bring-up and verification are done. The reached level is recorded by a new manifest revision in the repository (`releases/<tag>.toml`), never by moving the tag.
5. **Ratchet.** The target level of `rN+1` is at least the level `rN` reached (RFC 0010 rule 4); a row that passed on `rN` and fails on `rN+1` is a regression and blocks, whatever its level.
6. **Firmware, mechanics and assemblies** that are ordered or flashed as parts of a revision (case core, faceplate, battery pack) follow the same scheme with their own identity (`<identity>-rN`).
7. **Support.** A released hardware revision stays supported by current Blackbird OS, with no fixed time window: software adapts to the hardware it runs on (SYS-040), and an interface major that a released revision uses is kept (§10). Ending support for a revision is a maintainers' decision announced ahead with its date, never a silent drop.

### 4. Interfaces

1. **`<iface>/vN.M`.** `N` (major) changes when an implementation of the old version may not mate or work with an implementation of the new one (HAT interface v1.1 → v2, RFC 0014). `M` (minor) adds optional capabilities: a consumer written for `vN.K` works with any implementation of `vN.M` with `M ≥ K`. An interface written today as `v1` reads as `v1.0`.
2. **Owned once.** The component that owns the interface keeps its record (`interfaces/<iface>/`, [RFC 0018](0018-digital-thread.md) §2) and tags each version; tags live only there, and the devkit's compatibility matrix is generated from them, never holding tags of its own. Every implementation declares the versions it implements in its export.
3. **Frozen on first use.** A version may change while it is Draft; it is frozen when the first hardware revision or software release that implements it is tagged. After that, a change is a new version.
4. **Software interfaces** that hardware provides (`org.opendap.hw.*`, system hardware-support spec §7) follow the same rule: they only grow within a major version and are frozen at each release.

### 5. Exports and schemas

- **Exports.** `export/component.yaml` `version` is the producing component's tag (`composer/v0.3.0` or `t1-main-board-r1`); `name` is the component's path; `interfaces[].version` is `N.M`. A consumer pins each export it reads by that tag in its `consumes` list ([RFC 0018](0018-digital-thread.md) §3).
- **Schemas.** `N` rises only on an incompatible change; adding an optional key keeps `N` ([RFC 0018](0018-digital-thread.md) §3). A schema version is published with the component that owns the schema and never changes meaning after its first release.

### 6. Blackbird OS images

- Numbered by SemVer without codenames (Blackbird OS ADR 0011, OS-011). MAJOR when an interface version range the OS requires drops a major (system hardware-support spec §7), or when an update cannot be applied in place; MINOR for new capabilities; PATCH for fixes.
- One image per board (hardware-support spec §5): the image's manifest names the board support package and the board revisions and variants it was verified on.

### 7. The release manifest: `opendap.release/v1`

A TOML file written by the release command at `releases/<tag>.toml` in the releasing repository and attached to the published release. It is generated from records (tags, exports, variant files, verification matrices, check results); it is never written by hand, and the release command's `--check` regenerates it at the tag and compares.

```toml
schema = "opendap.release/v1"
tag = "t1-main-board-r1"
kind = "hardware-revision"          # software | hardware-revision | os-image | interface | product | devkit
date = "2026-11-02"
commit = "3f2a1c9e"

[[components]]                      # what the release contains, each pinned
name = "board"
tag = "t1-main-board-r1"
commit = "3f2a1c9e"

[[consumes]]                        # exports read, each at a tag
name = "faceplate-touch"
tag = "v0.4.0"

[os]                                # when the release ships or targets an image
version = "1.0.0"

[interfaces]                        # implemented or provided, N.M
faceplate = "1.0"
hat = "2.0"

[schemas]                           # versions read or written
"opendap.board-variant" = 1
"opendap.device-description" = 1
"opendap.check-result" = 1

[[variants]]
name = "full"
target_level = "Q1"
reached_level = "Q1"                # empty until verification is done
rows = { total = 0, passed = 0, failed = 0, unrun = 0, waived = 0 }
debt = []                           # verification row IDs above the target level, open
waivers = []                        # row IDs at or below the target, with the waiver's location (RFC 0010 rule 6)
evidence = []                       # check-result files collected for this variant and commit

[[artifacts]]                       # attached files
path = "t1-main-board-r1-fab.zip"
sha256 = "…"

changelog = "CHANGELOG.md"
```

The manifest is signed like the images (OS-011), with the same release key, so one key verifies everything a release ships; the signature is attached to the published release next to the manifest.

The manifest is what the developer compatibility matrix, the documentation and any later record read to know what a release contains; it is the end of the digital thread (requirement → verification row → check result → release).

### 8. Verifying a release

1. **`release` stage.** Every repository lists its release gate in the `release` stage of `check.toml` (`opendap.check/v1`). The release command runs it at the commit to tag and refuses to tag when it fails.
2. **Per variant.** For a hardware revision or a product release, each variant runs the rows of the verification matrices that its blocks touch, at its target level, on that revision's commit (RFC 0011 rule 9). Results are collected per variant; a result of one variant never counts for another. A variant that has not been verified is not listed in the manifest as released.
3. **Gate and debt.** Every row at or below a variant's target level passes or has a waiver; rows above it are listed as debt with their count (RFC 0010 rules 3 and 6).
4. **Software against every variant.** A Blackbird OS release runs its virtual-device suites against the generated device description of every variant it supports, and its device suites on every hardware revision it lists.
5. **Local.** Releases are built, checked and tagged on the developer machine (no CI, per the contributor guide); the manifest records the tool versions that produced the evidence (`opendap.check-result/v1` `tool.version`).

### 9. Changelogs

- `CHANGELOG.md` is generated per releasable component at release time from: the commits since the previous tag that touch the component's path, the ADRs accepted in that range, the interface and schema versions that changed, and the quality levels reached. It is never edited by hand; a hand-written note goes in the commit message.
- A commit that breaks a public surface says so in a `Breaking:` trailer, which the generator lists first and which the release command checks against the version bump (a `Breaking:` trailer without a MAJOR, or after 1.0 a MINOR, fails the gate).

### 10. Deprecation

| Surface | Stability (`opendap.tools/v1`) | Policy |
|---|---|---|
| Developer surfaces: the `opendap` CLI and its plugins, SDK crates, published schemas, package and theme formats, the component export contract, interfaces | `stable` | Deprecated with a warning on use (stderr and a `warnings` entry in `--json`) for at least one MINOR release; removed only in the next MAJOR; the changelog names the replacement (`replaces` in `tools.toml`). Readers keep accepting a superseded schema version for at least one MAJOR of the reader |
| Preview developer surfaces | `preview` | May change in a MINOR, with a changelog entry |
| Maintainer tools | `internal` | No compatibility promise: a change may land in any commit, provided every caller in the workspace is updated in the same change and the checks pass |
| Hardware interfaces | — | A major version stays supported by Blackbird OS while a released hardware revision implements it (§3 rule 7) |

Deprecated commands disappear from help and search but keep running until the next MAJOR (CLI spec A.11). A maintainer tool that developers start to depend on graduates to a developer surface first, and only then gets the `stable` promise.

### 11. Changes per repository

| Repository | Change |
|---|---|
| toolkit-host | The release command: run the `release` stage, collect results per variant, generate the manifest and changelog, create the annotated tag; `opendap.release/v1` schema with fixtures |
| toolkit-hardware | Hardware revision tags and fab packages named by revision; export `version` from the tag; `consumes` in the component record |
| toolkit-hooks | Commit-message check for `Breaking:` trailers |
| board, faceplate-touch, case, hat-standard, t1 | First hardware revision tags with target levels; interface records with `N.M` versions; `releases/` with manifests |
| system, blackbird-os | Image version scheme; images verified against each supported variant's description |
| cli, desktop | Existing SemVer and deprecation rules (CLI spec A.11) stated as instances of this RFC |
| rfcs | The template's "interface version" line refers to §4 |

## Alternatives

- **One version for the whole project, in lockstep.** Components move at very different rates (a board revision takes months, a tool changes daily); lockstep would make every tag meaningless for most components.
- **Calendar versions.** Say when, not what changed; Blackbird OS ADR 0011 and the user-facing conventions already chose numbers.
- **Plain `vX.Y.Z` tags in a shared repository.** Collide as soon as two components share it.
- **SemVer for hardware revisions.** A PCB revision has no API compatibility to express; what matters is which order it was and which level it reached.
- **Letter revisions (Rev A, B) or stage names (EVT, DVT, PVT).** Letters work but sort poorly past Z and carry no level; stage names describe maturity, which the quality levels already express (RFC 0010).
- **Hand-written release notes.** RFC 0010 needs debt counted, not described; only a generated manifest can be checked.

## Decisions

Accepted on 2026-10-27, each with its basis:

1. **Sub-revisions.** A PCB change is a new `rN`; a BOM or population change on the same PCB is `rN.M`; population options are variants (§3 rule 2). Basis: a revision is what was ordered (§3 rule 1), and the PCB is what an order fixes; population options are variants by RFC 0011 and BRD-053.
2. **No fixed support window.** A released revision stays supported by current Blackbird OS, and the interface majors it uses are kept; ending support is announced ahead with its date (§3 rule 7). Basis: SYS-040 (software adapts to the hardware it runs on).
3. **Release manifests are signed** like images, with the same release key (§7). Basis: OS-011 (images are signed, versioned and published as releases); one key verifies everything a release ships.
4. **Interface tags live only in the owning repository**; the devkit's compatibility matrix is generated from them (§4 rule 2). Basis: an interface is owned once (§4 rule 2; [RFC 0018](0018-digital-thread.md) §1 rule 3).
5. **The devkit's manifest is a release** of `kind = "devkit"` in `opendap.release/v1` (§7): one release format, one verifier. Basis: §7 already pins components, interfaces and schemas per release, which is what a devkit release lists.
