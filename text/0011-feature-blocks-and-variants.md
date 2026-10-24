# RFC 0011: Feature blocks and hardware variants

- **Status:** Accepted (2026-10-24). The intent was set by the maintainers: build the ideal prototype with every module and part wanted, down to the most specific ones (TPM, haptics), and keep a modular, reusable architecture that can enable, disable, change or modify any of them when something is not ready or not available, so the design stays resilient to change and integral; in simulation it serves comparison and co-design, in prototypes cost, performance, power, radio and thermal tuning, and in the product variants (lite versions, application-specific hardware, other chipsets or radios). The mechanics were decided by Claude under their delegation. Written by Claude.
- **Date:** 2026-10-24
- **Affected repositories:** board, faceplate-touch, hat-standard, kicad-lib, toolkit-hardware, toolkit-hooks, t1, system, virtual-devices.

## Summary

Every function of an OpenDAP board is a **feature block**: one hierarchical schematic sheet and one layout block, with a declared interface, power domain, safe state when absent, measurement point, alternates and software description. A **variant** is a named selection of blocks and alternates. The first prototype is the **full variant**: every block fitted and instrumented. Every other variant (production, lite, application-specific, another radio or chipset) is derived from it by disabling, swapping or re-parameterising blocks, never by redrawing the board's other blocks. The same block is used in the domain modules (integration level L2), in simulation and in every board that needs the function.

## Motivation

BRD-053 and board ADR 0013 already allow population options, and Blackbird OS already runs on hardware it describes rather than assumes (SYS-040, system hardware support spec, layers H0–H5). What is missing is the electrical discipline that makes a part optional in practice: a block left off must leave its buses, rails and straps in a defined state; a block swapped for an alternate must keep its interface; and each block must be measurable on its own, so power, thermal and radio behaviour can be tuned per block. Without it, an unavailable part or a late change forces a respin of unrelated circuitry, and integration results cannot be compared between variants.

## Proposal

### 1. Feature block

A block is the smallest unit that can be enabled, disabled, swapped or measured alone. Its record (one per block, in the board's block catalog) states:

| Field | Content |
|---|---|
| `id`, `function` | For example `blk.tpm`, key holder for the keystore (BRD-059) |
| `requirements` | Requirement IDs and their priority (must blocks are never optional in a product variant) |
| `interface` | The nets that cross the block boundary, by connector-style names (bus, address, interrupts, enables, clocks, supplies), as in the hardware-support connector rule |
| `power_domain` | The rail and the switch or enable that removes power from the block; the rail's measurement point |
| `absent_state` | What every interface net does when the block is not fitted or not powered: pull-ups or pull-downs, straps, series isolation, bus isolation; never a floating input or a back-powered part |
| `measure` | Current-sense position (shunt or 0 Ω link footprint), test points for the block's supply and signals |
| `alternates` | Same-land equivalents, alternate footprints (DNP) and their conditions; a swap that needs a new land is a block revision, not a board redesign |
| `software` | Devicetree node or overlay fragment and the component record of the device description (`opendap.device-description/v1`, H2), with `status = "okay"` driven by the variant |
| `sheet`, `layout_block` | The KiCad sheet and layout block, shared with the domain module and other boards |
| `level` | The RFC 0010 quality level the block has reached on its own |

### 2. Rules

1. **Full variant first.** Prototype 1 fits every block, including *should* parts and instrumentation (shunts, test points, isolation links), so every function is measured before any is removed.
2. **Absent is a designed state.** Every block passes the absent-state check: with the block unfitted, no net it touches floats, no IO is back-powered, no bus is held, no boot strap changes. The schematic gives the defaults; a tool checks them per variant.
3. **Blocks own their power.** Each optional block sits behind a switch or enable (load switch, regulator enable or 0 Ω link) and has a measurement point, so power, temperature and radio effects can be attributed per block and the block can be turned off in software or hardware.
4. **Boundaries are interfaces.** A block refers only to its interface nets. An alternate keeps the interface or declares the difference; the rest of the board does not change.
5. **Must blocks stay.** The audio chain, the power-safety margins (BRD-049) and every must requirement are fitted in every product variant (board ADR 0013); variants differ in should blocks, alternates and parameters.
6. **Variants are data.** A variant is a file listing blocks (`fit`, `dnp`, `alternate`) and parameters. The BOM, pick-and-place, devicetree overlays, the device description and the verification plan of each variant are generated from it. Nothing is edited by hand per variant.
7. **Software follows, never assumes.** Blackbird OS enables what the variant's description and the hardware's answers report (SYS-040, T1-045); a block that is absent is absent from the capabilities, not an error.
8. **Every variant is verified on its own.** Each variant carries its target quality level (RFC 0010) and runs the rows of the integration matrices that its blocks touch; results are never inherited from another variant.

### 3. Variants of the T1

| Variant | Purpose | Content |
|---|---|---|
| `full` | Prototype 1, the ideal prototype | Every block, every *should* part, every instrumentation footprint fitted |
| `production` | Commercial and DIY editions (T1-004) | Board ADR 0013 population; instrumentation links fitted as 0 Ω |
| `lite` (proposal) | A lower-cost T1 | *Should* blocks left off where they move the price materially (BRD-053 rule); the audio chain unchanged |
| `<application>` (as needed) | Application-specific hardware, another radio or chipset | Alternate blocks; a different SoC is a different H0/H1 board that reuses every block it can |

### 4. Uses

- **Simulation and co-design:** a block's L1 studies run per alternate and parameter set (`odapkit sim` checks per variant), so options are compared on the same criteria before parts are bought.
- **Prototypes:** blocks are enabled one at a time at bring-up; per-block current and temperature drive cost, power, radio and thermal tuning; a part that is late or fails is disabled without stopping the rest.
- **Product:** variants are built from data; each one is verified and certified on its own (T1-071), with the certified variant's transmitters limited to granted parts (T1-077).

### 5. Changes per repository

| Repository | Change |
|---|---|
| board | Block catalog for the T1 (`docs/specs/feature-blocks.md`) and variant files; schematic structured as one sheet per block; BOM variant fields per block |
| kicad-lib | Reusable blocks (sheets and layout blocks) for functions shared by several boards |
| toolkit-hardware | `odapkit variant`: per-variant BOM, pick-and-place, devicetree overlay and device-description generation; the absent-state check |
| toolkit-hooks | A hook that runs the absent-state check on changed schematics |
| t1 | Verification matrices per variant; the plan's variants |
| system | The device description's component records carry the block identity |
| virtual-devices | Virtual variants from the same variant files, so software is tested against each |
| faceplate-touch, hat-standard | The same block discipline on their boards |

## Alternatives

- **Population options only (BRD-053 as it was).** Leaves absent states, power attribution and software enablement to chance.
- **One board per variant.** Multiplies layouts and loses comparability between variants.
- **Hardware modules on connectors inside the product.** Raises size, cost and failure points in a handheld; kept for the L2 domain modules, not for the product.

## Open questions

- Whether KiCad 10's own variant support covers rule 6, or the variant files drive a generator around it (**UNVERIFIED**).
- How far layout blocks can be shared between projects in KiCad 10 (also open in the integration and verification spec).
