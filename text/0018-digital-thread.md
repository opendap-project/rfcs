# RFC 0018: The digital thread: records, owners and schemas

- **Status:** Accepted (2026-10-27). The records are applied one document at a time (§4 rule 4); the repository changes in §8 are listed, not applied.
- **Date:** 2026-10-27
- **Affected repositories:** toolkit-host, toolkit-hardware, toolkit-verify, toolkit-hooks, rfcs, t1, board, case, faceplate-touch, hat-standard, parts-lib, kicad-lib, system, virtual-devices; later every component repository. The names are today's; [RFC 0017](0017-naming-and-topology.md) groups them into roots, which changes paths, not this RFC.

## Summary

Every fact that a tool or another component reads is a **record**: a data file with a published schema named `opendap.<kind>/vN`, one owner, one generator for each file derived from it and one check that fails when a derived file is stale or a reference does not resolve. Together the records form one chain, the **digital thread**, from a requirement to the block and variant that meet it, the parts and BOM that build it, the device description that Blackbird OS reads, the verification rows and check results that prove it, the component export that others consume, and the release that ships it. Markdown that restates a record is generated from it and checked (`--check`), never kept by hand. Records flow from public to private only. The proposal builds on what already exists: quality levels (RFC 0010), feature blocks and variants (RFC 0011), the component export contract, `opendap.board-variant/v1`, `opendap.device-description/v1`, and the workspace schemas `opendap.check-result/v1`, `opendap.matrix/v1`, `opendap.check/v1` and `opendap.tools/v1`. It also gives every level scale its own letter (Q, I, V, T, C, H), so that `L2` stops meaning five different things.

## Motivation

- **Facts kept as prose drift.** Pinouts, the power tree, the BOM, the block catalog and the verification matrices are Markdown tables today. Each copy (the faceplate connector pinout is restated in several repositories) is edited by hand, and nothing fails when two copies disagree. RFC 0011 rule 7 already requires the BOM, pick-and-place, devicetree overlays, the device description and the verification plan of each variant to be generated from the variant file; most of their inputs are not data yet.
- **Traceability stops at the document.** Requirements cite IDs and verification matrices cite requirements, but a check result does not point back at the row it fills, and no tool can say which must requirement of a variant is unverified at its target level (RFC 0010 rule 3, RFC 0011 rule 9).
- **Software must follow the hardware** (SYS-040, T1-045). The device description the OS reads must come from the same data as the board that is built, or the two diverge on the first variant.
- **Scale and endurance** (T1-046, T1-047) apply to the project's own data too: with dozens of components and variants, only checked, generated records stay consistent.
- **Level names collide.** `L0`–`L6` means integration levels (t1), `L0`–`L5` software verification levels (toolkit-verify), `L0`–`L4` test levels (system ADR 0014), `L1`–`L3` HAT compatibility levels (hat-standard ADR 0003) and `L0`–`L7` package layers (system ADR 0011). A check result that says `level = "L2"` cannot be read without knowing which document wrote it; RFC 0010 already had to call itself "orthogonal to the integration levels L0–L6 and to the HAT compatibility levels L1–L3".

## Proposal

### 1. Principles

1. **Data first, prose generated.** A fact that a tool or another component reads is data with a schema. Its Markdown is generated from the data and checked; it is never the source.
2. **One engine, many surfaces.** The logic that validates, generates or checks a record lives in one tested library. The maintainers' command, the pre-commit hook, the `check.toml` step and any agent procedure are thin wrappers around it, so the same rule gives the same answer wherever it runs.
3. **One owner per record.** A record has one owning component and one location. Every other component consumes it, through the owner's export or by reference to a tagged version, and never keeps a copy.
4. **Public to private only.** Public records never depend on private ones (§5).

### 2. Records

| Record | Owner and location | Schema | Generated from it | Check | Consumed by |
|---|---|---|---|---|---|
| **Requirements** | Each component: `docs/requirements/requirements.toml` | `opendap.requirements/v1` | `docs/requirements/README.md`; the workspace requirement index | Schema; IDs unique and never renumbered; prefix registered in the workspace prefix registry; every cited ID resolves (requirement-reference hook, on every Markdown and data file) | Blocks, variants (`promise`), verification rows, check results, the release manifest, docs |
| **Interface** | The component that owns it: `interfaces/<iface>/interface.toml` plus its data (pinout, protocol, mechanical envelope) | `opendap.interface/v1` | Pinout and signal tables in the owner's spec and `docs/reference/`; the connector map of each module and board that implements it | Schema; version rule ([RFC 0019](0019-versioning-and-releases.md) §4); every implementing netlist matches the interface (rule M3 of the electronics integration method, generalised) | Blocks (boundary nets), component exports (`interfaces`), device descriptions (`provides`), HAT and faceplate conformance suites |
| **Block** | Board, faceplate or HAT: `blocks/<block>.toml` | `opendap.block/v1` | The block catalog (`docs/reference/blocks.md`); the sheet skeleton for new blocks | Schema with the fields of RFC 0011 §1; interface nets exist on the block's sheet; requirement IDs resolve; declared `params` and `alternates` are complete; the absent-state check (RFC 0011 rule 2) | Variants, BOM, device description, verification plan |
| **Variant** | The board (or faceplate, HAT): `variants/<variant>.toml` | `opendap.board-variant/v1` (exists, board feature-blocks spec §4.1) | KiCad variant data, BOM, pick-and-place, devicetree overlay, device description, per-variant verification plan (RFC 0011 rule 7) | Schema; every catalog block exactly once; every block bound by a must requirement of `promise` is `fit` or `alternate`; an `open` entry forbids `accepted`; a `dnp` block passes its absent-state check | Every per-variant record below; the release manifest |
| **BOM** | Generated per variant: `bom/<variant>.json` and `docs/reference/bom-<variant>.md` | `opendap.bom/v1` | Fab package BOM and pick-and-place files | Generated is current; every line resolves to a part record; every part's lifecycle status is acceptable at the variant's target level | Fab packages, the release manifest, private records (read only, §5) |
| **Part** | parts-lib: `<manufacturer>/<mpn>.md` with TOML front matter | `opendap.part/v1` | The library manifest and index | Schema; one manufacturer key (lowercase kebab-case ASCII) in every library; every value cites its source; `checked` date present; lifecycle field present | BOM, kicad-lib land data, selection matrices, private sourcing |
| **Device description** | Generated per variant for a board; per faceplate and HAT from their own data | `opendap.device-description/v1` (defined in the system hardware-support spec §3) | The board support package's description file; virtual-device fixtures | The Blackbird OS validator, the same one the image build runs; generated is current | Blackbird OS (board support package), virtual devices, the UI |
| **Verification matrix** | Each component: `verification/matrix.toml` | `opendap.verification/v1` | The matrix tables in the component's spec and `docs/reference/`; the per-variant verification plan | Schema; every must requirement of every variant's `promise` has at least one row; every row names a check that exists in a `check.toml`; status is only ever filled by collection (§4) | Release gate, debt report, docs |
| **Check result** | Every tool, test runner and evaluation | `opendap.check-result/v1` (exists) | Verification row status; audits | Schema; `subject.commit` present for anything cited as evidence | Collection (§4), audits, the release manifest |
| **Evaluation matrix** | Each evaluation or selection: `studies/<domain>/<study>/` | `opendap.matrix/v1` (exists) | Its report | Schema; every value cites a source with its status; a recommendation is labelled as one | Block `alternates`, ADRs, verification rows that cite study results |
| **Component export** | Every component: `export/component.yaml` | `opendap.component/v1` (exists as `component.schema.json`, without a schema key yet) | The export files themselves, from the sources | `component validate`; every consumed export pinned to a tag (§3) | Other components, the product integration, the release manifest |
| **Release** | Each release of a root or component | `opendap.release/v1` ([RFC 0019](0019-versioning-and-releases.md) §7) | Release notes and changelog | Every gate item at or below the target level passes or carries a waiver | Users, the developer compatibility matrix, private records |

Two workspace schemas are not records of the thread but run it: `opendap.check/v1` (a root's `check.toml`: which generator and check runs at which stage, which requirement IDs a step `verifies`, whether it emits a check result) and `opendap.tools/v1` (a root's `tools.toml`: how the commands that wrap the engines are found). The maintainers' planning formats (task and decision files) carry no product facts and are outside the thread; they are tool formats, not contracts, and keep plain schema names (`task`, `decisions`).

### 3. Schemas

- **Name.** A schema that crosses domains is `opendap.<kind>/vN`; a schema internal to Blackbird OS is `blackbird.<kind>/vN`. `<kind>` is lowercase kebab-case and names the record, not the tool.
- **Every record says its schema.** Each file carries `schema = "opendap.<kind>/vN"` (TOML) or `schema: opendap.<kind>/vN` (YAML, JSON). The component export gains this key in its next schema revision; until then a file without the key is read as `opendap.component/v1`.
- **Where schemas live.** Each schema is a JSON Schema 2020-12 file in the repository of the owning domain, with good and bad fixtures and a spec section, changed together. One home per owner: the maintainers' tool formats in the tools' schemas (toolkit-host and toolkit-hardware today, `tools/schemas` after the `tools` cut), Blackbird OS formats with the OS (the device description in system today, `blackbird/schemas` after its cut), and the contracts developers build against in the devkit, the component export contract and `opendap.component/v1` included (RFC 0017 §2). One generated catalogue indexes every schema under one URL space, and each published schema has a stable URL there; there is no separate schema repository.
- **Versions.** `N` rises only on an incompatible change (a removed or renamed key, a narrowed type, a changed meaning). Adding an optional key keeps `N`. Readers accept every version they document and reject unknown versions with a clear error. A tool that writes records writes the newest version it knows.
- **Unknown keys are rejected**, as the component export schema does today: a new field is added to the schema first.
- **Consumption is pinned.** A component lists what it consumes (`consumes`: export name, tag) in its component record; a consumer reads only the consumed tag's `export/`, never the producer's sources (component export contract, rules 1 and 2).

### 4. Prose from data

1. **Source and output.** The data file is the source. Each generated Markdown file starts with `<!-- @generated by <tool> from <source>; do not edit -->`. A generated table inside a hand-written spec sits between `<!-- BEGIN GENERATED <tool> <source> -->` and `<!-- END GENERATED -->` markers; the text outside the markers stays hand-written.
2. **`--check`.** Every generator has a `--check` mode that renders in memory, compares, and exits 1 with the file name when the committed output differs. Each root lists these checks in the `fast` stage of its `check.toml`, so they run before every commit.
3. **Where generated docs go.** Whole generated documents (schemas, block catalog, BOM, pinouts, matrices, command help) go in `docs/reference/`. Specs keep the rules and the reasoning and embed only the generated tables they need.
4. **Migration, one document at a time.** A prose specification of facts becomes data in one change: the data file is written from the existing table, the generator renders it, the rendered table is compared with the hand-written one (parity), and only then is the hand-written table replaced by the generated block. Values that the prose marked **UNVERIFIED** keep that status as a field.
5. **Copies become references.** A fact stated in several repositories (a connector pinout stated by its owner and its consumers) is kept once, by the owner of the interface; the consumers render it from the owner's tagged export.

### 5. Public and private records

- Public records, schemas and tools never read a private record, never carry a field whose value exists only privately, and never fail because private data is absent. A public checkout builds, generates and checks completely.
- Private tools may read public records at a tagged version (for example a generated BOM or a released unit's records). What they produce stays private and is never written back into a public record.
- Where a public record needs a fact that comes from private work (an availability check, a decision), it states the fact as a value with its date ("Sourcing check (T1-029): passed, 2026-10-26"), never a reference to the private source.

### 6. Traceability: requirement to release

```text
requirement (ID, priority)
  └─ verification row (verifies: [ID], level, variants, check, method, negative control)
       └─ check step in check.toml (name, verifies: [ID], emits: check-result)
            └─ check result (results[].verifies, level, subject.repo, subject.commit, subject.variant, evidence)
                 └─ collection fills the row's status per variant and commit
                      └─ release manifest: per variant, level reached, debt, waivers
```

1. **A row names what it verifies and where.** Each verification row carries the requirement IDs it verifies, its level (Q for quality, I, V or T for where it runs, §7), the variants it applies to, the check that produces its evidence, and, for checks verified by software, the negative control that must fail (toolkit-verify verification spec §1).
2. **Results are emitted, never typed.** Every check that is cited as evidence emits `opendap.check-result/v1` with `verifies`, `level`, `subject.commit` and, for hardware, `subject.variant`. A collection command fills the rows' status from those results; nobody edits a status by hand.
3. **A result belongs to one commit and one variant.** It is never inherited by another variant (RFC 0011 rule 9). A result goes stale when the commit changes an input of its row; regression follows the dependency graph of the integration matrix (a changed block reruns its rows and the rows of every interaction it takes part in).
4. **Gaps are computed.** For each variant and its target level, the collection lists the must requirements without a passing row at or below that level (blocking), the rows above it (debt, RFC 0010 rule 3) and the waivers (RFC 0010 rule 6). The release manifest records the result.

### 7. Level scales

Every scale gets its own letter. Numbers inside each scale stay as they are, so existing documents change only their prefix.

| Letter | Scale | Today | Defined in |
|---|---|---|---|
| **Q** | Quality level: what a build meets | Q0–Q4 (unchanged) | RFC 0010 |
| **I** | Integration level: how much of the product is integrated | L0–L6 → I0–I6 | t1 electronics integration and verification spec |
| **V** | Software verification level: where a check runs (static, model, virtual, development hardware, prototype, accredited lab) | L0–L5 → V0–V5 | toolkit-verify verification spec; `opendap verification list --level` |
| **T** | Test level of Blackbird software (unit, property, interface, end to end, device) | L0–L4 → T0–T4 | system ADR 0014 |
| **C** | HAT compatibility level | L1–L3 → C1–C3 (historical: hat-standard ADR 0006 removed compatibility levels from interface v2; the letter keeps old citations unambiguous) | hat-standard ADR 0003 |
| **H** | Hardware-support layer | H0–H5 (unchanged) | system hardware-support spec, system ADR 0013 |

After the change, `L0`–`L7` means only the package layers of Blackbird OS (system ADR 0011), which are layers, not levels. Records write a level as `<letter><n>`; `opendap.check-result/v1` already accepts `^[QIVCH][0-9]+$` and gains `T`. Accepted ADRs are not edited: the specs that use the scales are updated, and ADR 0014's levels are restated as T0–T4 in the spec that applies them.

### 8. Changes per repository

| Repository | Change |
|---|---|
| toolkit-host | Workspace schemas gain `T` in the level pattern; the data-to-Markdown renderer with `--check` and generated-block markers; the requirement index and requirement-reference check; collection of check results into verification matrices |
| toolkit-hardware | Schemas `opendap.interface/v1`, `opendap.block/v1`, `opendap.bom/v1`, `opendap.verification/v1`; `opendap.board-variant/v1` moves here from the board spec as a JSON Schema with fixtures; `variant` generates BOM, pick-and-place, devicetree overlay, device description and verification plan; the component export schema gains `schema` and `consumes` |
| toolkit-verify | Checks emit `opendap.check-result/v1` with `verifies` and V levels; the traceability table becomes a verification matrix record |
| toolkit-hooks | Hooks for schema validation, generated-is-current and requirement references, wrapping the engines above |
| t1, board, faceplate-touch, case, hat-standard | Requirements, interfaces, blocks, variants and verification matrices as records; their prose tables become generated blocks, one document at a time (§4 rule 4); integration levels renamed I0–I6 |
| parts-lib, kicad-lib | Part front matter as `opendap.part/v1`; one manufacturer key |
| system | Publishes the `opendap.device-description/v1` schema and validator; test levels written T0–T4 in the specs |
| virtual-devices | Reads generated device descriptions as fixtures, one per variant |
| rfcs | RFC 0010's note on orthogonal scales and its open question refer to I and C levels |

This RFC is a contract between repositories; it changes no interface version.

## Alternatives

- **Keep prose specs and review them carefully.** Has not prevented copies from diverging; no tool can compute a gap from prose.
- **One central database.** A single store outside the repositories would break "consumers read only `export/`", tie every component to one service and hide history; records in git keep history, review and tags.
- **Generate everything from KiCad.** KiCad holds the electrical design, not requirements, absent states, software descriptions or verification; and ERC, DRC and the netlist take no variant (RFC 0011, open questions).
- **One schema per document** instead of per record kind. Multiplies formats that tools must learn; a record kind is reused by every component.
- **Rename only the colliding level scales.** Leaves `L` with two meanings; each scale having its own letter costs one prefix change per document.

## Decisions

Accepted on 2026-10-27, each with its basis:

1. **The component export contract lives in the devkit.** The contract and `opendap.component/v1` are what developers build against; the maintainers' tools and the `opendap` CLI consume them (§3). Basis: the audience rule of RFC 0017 §4 (developer contracts never depend on maintainer tools).
2. **One schema home per owner, one catalogue.** Tool formats with the tools, OS formats with Blackbird OS, developer contracts in the devkit, all indexed by one generated catalogue under one URL space; no separate schema repository (§3). Basis: §1 rule 3 (one owner per record) and RFC 0017 §2's roots.
3. **Faceplate and HAT descriptions are generated** from their own block and variant records, as for a board; a hand-written description is allowed only until the component has block records, and the schema check applies to both. Basis: RFC 0011 asks faceplate-touch and hat-standard for the same block discipline as the board.
4. **`opendap.verification/v1` references results.** It references the collected `opendap.check-result/v1` documents by path and commit and keeps only the derived observed state per row and variant, so a result is never copied twice. Basis: §6 rule 2 (results are emitted, never typed).
5. **Planning formats keep plain names** (`task`, `decisions`): they are maintainers' tool formats, not contracts others build against (§2).
