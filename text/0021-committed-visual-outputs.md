# RFC 0021: Committed visual outputs

- **Status:** Accepted (2026-10-29, maintainer approval). Implemented in tools (`odw render`, the `outputs-current` and `outputs-regenerate` hooks, the `no-stray-files` exception) and in t1's board; the other repositories take it as they gain sources (§9).
- **Date:** 2026-10-29
- **Affected repositories:** tools (the `render` tool and the hooks), workspace (the templates block that rolls the hooks out), t1 (the board's outputs first; case and faceplates when they have sources), hats, libs (when they declare outputs)

## Summary

Every repository that holds design sources (KiCad schematics and boards, FreeCAD and Blender models) commits a fixed set of deterministic visual outputs next to them: PDF and SVG schematics, per-layer SVG plots and 3D renders of each board, PNG views and glTF of each model. They sit in an `outputs/` folder beside the component's `export/`, declared by an `outputs.toml`, so GitHub shows the design without KiCad, FreeCAD or Blender. They are generated locally by one tool, `odw render`, with no agent in the loop; a source-hash stamp makes staleness cheap to check before each commit, and a pre-push hook regenerates them and fails on a difference. They are never fabrication outputs: Gerbers, drill and placement files stay out of the repositories.

## Motivation

- **A design is unreadable on GitHub without its tools.** A `.kicad_sch` or `.FCStd` is text or a zip archive; a reviewer or a contributor without KiCad 10 or FreeCAD sees nothing of the T1 main board's 69 sheets or its placement. The public repositories exist so that others can build, use, modify, repair and reproduce what OpenDAP publishes (contributor guide, Licensing and Public and private repositories).
- **Local only, no CI** (contributor guide, Local only): checks and generated outputs run on the developer machine, so outputs are generated there and committed, as the exports are (component export contract rule 4).
- **Fab outputs stay out.** The `no-fab-outputs` hook rejects Gerbers, drill and placement files; fab packages are built locally and attached to releases. Visual outputs are a different set and must not reopen that door.
- **PDFs are rejected today.** The `no-stray-files` hook rejects every PDF, so manufacturer documents are not redistributed; a schematic PDF of our own design needs a declared, narrow exception.
- **Staleness must be cheap to see.** Regenerating a 69-sheet schematic and a 996-part board takes tens of seconds; a check before every commit cannot regenerate, it must compare hashes.

## Proposal

### 1. Scope

1. Every public repository with a source type of §3 declares its visual outputs in an `outputs.toml` and commits them. A repository without such sources has none.
2. Visual outputs are for people: they are not an interface. Consumers keep reading only `export/` (component export contract rule 1); no tool reads another repository's `outputs/`. The component export contract does not change.
3. Fabrication outputs (Gerbers, drill, pick-and-place, IPC-2581, ODB++, STEP for manufacturing) are never visual outputs; `no-fab-outputs` keeps rejecting them everywhere, `outputs/` included.
4. Generation is local and deterministic (§5), run by `odw render` or the hooks (§6), with no agent in the loop: the same sources and tool versions give the same files.

### 2. The output folder

1. A component's outputs live in `outputs/`, beside its `outputs.toml`, at the component's root: the folder that holds `export/` when it has one (`t1/board/outputs/`, `t1/case/outputs/`, `hats/<hat>/outputs/`).
2. Why not inside `export/`: the export is the consumers' interface, committed with the producer's tag (rule 4) and listed in `component.yaml`; visual outputs change with every source change and are read by people. Putting them in `export/` would make every schematic edit an export change and widen the interface. A file both sides need (a glTF of a model) is generated once per side: the export step writes the export, `odw render` the view.
3. The folder name is fixed, never configured: tools, hooks and reviewers find it without reading a manifest.
4. Layout, one subfolder per declared source, named after it (`name`, default the source file's stem):

| Path under `outputs/` | What |
|---|---|
| `README.md` | Generated index: each source, its outputs, the renders inline |
| `stamp.json` | The source-hash stamp (§4) |
| `<name>/schematic.pdf` | Every sheet, one PDF |
| `<name>/schematic/<sheet>.svg` | One SVG per sheet, named after the sheet path in lower case |
| `<name>/pcb/<layer>.svg` | One SVG per plotted layer (`F_Cu.svg`), each with the board outline |
| `<name>/render/<view>.png` | 3D renders from fixed views |
| `<name>/<name>.glb` | glTF (binary), where declared |

### 3. Output set per source type

`outputs.toml` (`opendap.outputs/v1`) lists the sources; each `[[source]]` has a `type`, a `path` relative to the component and the options below. A KiCad source may name a colour theme file (`kicad_theme`), read from the component's own tree (the libs submodule's `kicad/colour-themes/opendap.json` for the T1 board).

| `type` | Source | Outputs | Generated with | Options (default) |
|---|---|---|---|---|
| `kicad-sch` | root `.kicad_sch` | `schematic.pdf`, `schematic/<sheet>.svg` | `kicad-cli sch export pdf`, `sch export svg` | `kicad_theme` |
| `kicad-pcb` | `.kicad_pcb` | `pcb/<layer>.svg`, `render/<view>.png`, optional `<name>.glb` | `kicad-cli pcb export svg --mode-multi`, `pcb render`, `pcb export glb` | `layers` (every copper layer, F/B.Silkscreen, F/B.Fab), `views` (`top`, `bottom`, `iso-front`, `iso-back`), `size` (1600 × 1200), `glb` (false), `kicad_theme` |
| `freecad` | `.FCStd` | `<name>.glb` | `freecadcmd`: glTF through FreeCAD's exporter; PNG views need FreeCAD's 3D view, which `freecadcmd` does not have, so `views` stays empty (open question) | `glb` (true) |
| `blender` | `.blend` | `render/<view>.png`, `<name>.glb` | `blender --background`: the Workbench engine for PNG, the glTF exporter | `views` (`front`, `top`, `iso`), `glb` (true), `size` (1600 × 1200) |

No STEP-to-glTF step is needed: KiCad exports glTF from the board, FreeCAD and Blender from their models. An adapter whose program is missing reports `render.unavailable` for its outputs and fails the run; `odw render list` shows which adapters a machine has.

### 4. Source-hash stamps

1. `outputs/stamp.json` (`opendap.outputs-stamp/v1`) records, for each source: its type and options (hashed), the SHA-256 of every input, the SHA-256 of every output, and the program versions that generated them (render format version, KiCad version).
2. **Inputs** are the files the outputs depend on: for `kicad-sch`, every `.kicad_sch`, `.kicad_pro` and `.kicad_wks` in the source's folder and the theme file; for `kicad-pcb`, the board, its `.kicad_pro`, the theme file and every 3D model the board references, by its variable path (`${OPENDAP_KICAD_3DMODELS}/…`); for `freecad` and `blender`, the model file and the files it links.
3. **Current** means: every input hash and the options hash equal the stamp's, every output exists with the stamp's hash, and `outputs/` holds no file the stamp does not list. Checking it reads and hashes files only; it never runs KiCad.
4. A program version is recorded, not compared: a maintainer with another KiCad still passes the commit check; the pre-push regeneration (§6) shows whether that version changes the files.

### 5. Determinism

1. **Isolated program settings.** KiCad runs with a fresh `KICAD_CONFIG_HOME` holding only the declared theme, so a maintainer's own colours, fonts or plot settings never reach an output; Blender runs with `--factory-startup`.
2. **No timestamps, paths or random identifiers.** The tool rewrites what the programs stamp: the PDF `CreationDate` (same length, so the cross-reference offsets hold), the date in each SVG title; PNG files are written without ancillary chunks (`tIME`, `tEXt`).
3. **Fixed views and sizes**, from `outputs.toml`, never from the program's last session.
4. **Text outputs are byte-identical** across runs (PDF, SVG, README, stamp). KiCad 10.0.7's 3D renderer is not: repeated renders of the T1 board, 1568 × 1176, differed in 28 to 58 pixels (0.003 %) from the top and bottom and in 570 to 758 pixels (0.041 %) from the isometric views, by at most 17 in a channel. A regenerated PNG that differs from the committed one in at most 0.1 % of its pixels, by at most 32 in a channel, counts as unchanged and the committed file is kept; beyond that it is a change.
5. **SVG size.** KiCad draws each stroke-font glyph segment as its own `<path>`; the tool merges the paths of each glyph group into one path of several subpaths, which draws the same and takes about a quarter less (90.8 MB to 67.2 MB over the T1 board's 69 sheet and 10 layer SVGs). Paths of filled shapes are never merged.

### 6. Hooks

From the tools repository (`.pre-commit-hooks.yaml`), pinned by tag:

| Hook | Stage | What it does | On failure |
|---|---|---|---|
| `outputs-current` | pre-commit | `odw render check`: every component's stamp is current (§4.3) | Names the stale component and the command: `odw render update <component>` |
| `outputs-regenerate` | pre-push | Regenerates the components whose sources or outputs the push changes, then fails when `outputs/` differs from the commit | Names the changed files: commit them (`git add <component>/outputs`) and push again |

Both run from the pinned tools checkout (`hardware/tools/render`), so a repository needs no `odw` on its `PATH`; they need Python 3.11 and the programs of the declared sources (KiCad 10 for KiCad sources).

### 7. Fab outputs and stray files

1. `no-stray-files` allows a PDF only inside an `outputs/` folder that has an `outputs.toml` beside it and whose `stamp.json` lists that file: our own generated schematic. Every other PDF is still rejected (manufacturer documents are not redistributed). HTML keeps its rule (standalone 3D previews under `renders/web/`).
2. `no-fab-outputs` is unchanged and applies inside `outputs/` too; no visual output uses a fabrication extension.

### 8. Sizes and LFS

1. **Per file: 5 MB**, the limit `check-added-large-files` already sets; **per component: 100 MB** in `outputs/`. `odw render update` fails on either, naming the file; a source over them reduces its `layers`, `views` or `size`.
2. SVG, PDF, PNG and README are plain git objects: GitHub shows them inline, they compress well (a 3.1 MB sheet SVG is 0.43 MB gzipped) and git deltas successive versions. An unchanged sheet regenerates to the same bytes and adds nothing to history.
3. glTF (`.glb`) follows the LFS rule the templates already set for CAD binaries (`templates/gitattributes/lfs`); the T1 board's glTF (6.7 MB) is over the per-file limit, so `kicad-pcb` sources leave `glb` off by default.
4. `outputs/` is marked `linguist-generated` (`.gitattributes` fragment), so GitHub folds it in diffs and language statistics.

### 9. Rollout

1. The workspace templates add a fragment to the `.pre-commit-config.yaml` block: the two hooks, pinned at the tools tag that ships them, rendered for a repository that tracks an `outputs.toml` (`when_tracked`); and a `.gitattributes` fragment for `outputs/`. `odw sync --apply` writes them; a repository opts in by committing its `outputs.toml` and its first outputs.
2. First: t1's board (the T1 main board schematic and board). Then each repository as it gains sources: t1's case and faceplates and the HATs (FreeCAD and Blender), libs (its KiCad examples and design-rule test boards, which fail DRC by design and are not visual outputs unless declared).
3. Each affected repository records the outcome in its own `docs/` or `README.md` (RFC process, step 4).

## Alternatives

- **Outputs inside `export/`.** Rejected (§2.2): it makes visual files part of the consumer interface and ties them to tags.
- **CI-generated outputs** (KiBot, GitHub Actions artefacts). Rejected: no CI (contributor guide, Local only), and artefacts are not shown in the repository view.
- **Regenerate on every commit.** Rejected: tens of seconds per commit for a board; stamps make the commit check a hash comparison and move regeneration to the push.
- **LFS for every binary output.** Rejected for PDF and PNG: LFS objects count against a bandwidth quota and add a client step for every contributor, for files of a few hundred kilobytes; kept for glTF, as for other CAD binaries.
- **Byte-identical PNG renders.** Not reachable with KiCad 10.0.7's renderer (§5.4); the tolerance keeps committed renders stable instead.

## Open questions

- **Raster tolerance across KiCad versions.** The 0.1 % / 16 tolerance is measured on one machine and one KiCad version; a KiCad update that changes lighting regenerates every render, as intended.
- **FreeCAD PNG views.** `freecadcmd` has no 3D view; a FreeCAD source gets glTF only until a headless view is available (for example rendering its glTF through Blender's Workbench engine).
