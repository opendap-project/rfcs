# RFC 0010: Quality levels Q0–Q4, incremental and ratcheted

- **Status:** Accepted (2026-10-24).
- **Amended by:** [RFC 0018](0018-digital-thread.md) (§7: Q stays the quality scale; the other level scales get their own letters, I, V, T, C and H, so the HAT compatibility levels L1–L3 are written C1–C3)
- **Date:** 2026-10-24
- **Affected repositories:** t1, board, faceplate-touch, hat-standard, kicad-lib, parts-lib, toolkit-hardware, toolkit-hooks, toolkit-verify (later every hardware repository).

## Summary

Every rule, check, verification row and certification item OpenDAP applies to its hardware carries the lowest **quality level** at which it becomes mandatory: Q0 (bench-safe) to Q4 (reference). Each build or release declares the level it targets. It must pass every item at or below that level; items above it are recorded as visible debt, not blockers. A level once reached is never lost: later revisions may raise it, never lower it.

## Motivation

The project now has many sources of demands: PCB design constraints (BRD-060), fab and product DRC (KLIB-002), the craft standard, industry practice, the integration and verification matrices, and certification (design for compliance). Applied as one flat list they either block every prototype on the last item or, relaxed, lose track of what is missing. The maintainers asked (2026-10-24) for a framework from the minimum to the highest aim, with incremental improvement rather than blocking perfectionism. T1-048 already sets three thresholds per experience metric (ideal, warning, hard limit); this RFC extends the idea to everything the hardware is held to, and is orthogonal to the integration levels L0–L6 (where something is evaluated) and to the HAT compatibility levels L1–L3 (what a HAT implements).

## Proposal

### 1. Levels

| Level | Name | Meaning | Typical use |
|---|---|---|---|
| **Q0** | Bench-safe | Safe to power on a bench by its developers: protections in place, no fab-rule or electrical-rule error, battery handled under its safety rules | Coupons, first module boards |
| **Q1** | Prototype | Works as designed and can be handed to other developers: product DRC errors at zero, the evaluation rows of its integration level pass, every warning recorded | Modules, prototype 1 |
| **Q2** | Pre-compliance | Fit to leave the lab in small numbers: every hard limit of the product's experience thresholds met (T1-048 for the T1), pre-compliance scans pass with margin, ESD and acoustic output measured, pack transport-safe | Pilot units, kits |
| **Q3** | Compliant | A product: every mandatory certification for the declared markets passed, technical file complete, every warning threshold cleared | Products on sale |
| **Q4** | Reference | The highest aim: ideal values of every threshold, full craft standard, wide margins, every warning and debt item closed | The goal every revision moves toward |

Products that set three thresholds per metric (ideal, warning, hard limit; T1-048 for the T1) map them onto the levels: hard limits are mandatory from Q2, warnings from Q3, ideal values at Q4. Safety limits (BRD-049) are mandatory from Q0.

### 2. Rules

1. **Every item carries its level.** A rule, check, checklist item, verification row or certification item states the lowest level at which it is mandatory, written `[Qn]` next to its ID. An item without a level is Q1.
2. **Each build declares a target level.** A fab release, a module, a prototype or a product release names its target in its release notes.
3. **Gate at the target, debt above it.** Every item at or below the target passes, or the build does not go out. Items above the target are reported as debt: listed, counted and visible, never silently dropped.
4. **Ratchet.** Once a component reaches a level, later revisions keep it. A change that makes a passing item fail is a regression and blocks, whatever its level.
5. **Raise one step at a time.** A revision plan names the debt it closes; the level rises when the next level's items all pass. No revision waits for Q4 to ship at its own level.
6. **No silent waivers.** An item at or below the target that cannot pass gets a written waiver in the component's docs: reason, risk, the revision that closes it, and the maintainers' sign-off. Waivers count as debt.

### 3. Tools

- **DRC.** KiCad rule severities are static, so rule names carry the level (`T1 PCB-G-01 [Q1]: …`). Rules at Q0–Q1 are errors; rules at Q2–Q4 are warnings. `odapkit` reports group violations by level and compute the level a board reaches.
- **Checks and simulations.** `odapkit sim` checks and the craft and layout scripts carry a `level` field; reports group results by level.
- **Verification matrices** (t1 electronics integration and verification) tag each row with its level; first-article and bring-up reports state the level reached.
- **Debt.** Each component keeps its open items above its current level in its docs (a debt table in the relevant spec), ordered by the level they unlock.

### 4. Changes per repository

| Repository | Change |
|---|---|
| t1 | Electronics integration and verification: level per row. Compliance requirements (T1-071 to T1-086): level per item (mostly Q2–Q3). Plan: each fab release names its target level. |
| board | PCB design constraints and product rules: level per constraint and rule; prototype 1 targets Q1. |
| kicad-lib | Fab rule sets are Q0. Craft standard: level per CRAFT rule. |
| toolkit-hardware | `odapkit` reports by level (dru, sim, craft check); review findings carry levels. |
| toolkit-verify | Conformance suites tag cases with levels. |
| faceplate-touch, hat-standard | Same tagging when their constraints and rules are written. |

## Alternatives

- **One flat list, everything mandatory.** Blocks early prototypes on items that only matter for a product, and invites ignoring the list.
- **Per-document severities without a shared scale.** Each document would invent its own; a build could not say what it meets.
- **Levels without a ratchet.** Allows quiet regression, against T1-047's spirit of no degradation over time.

## Open questions

- The level of each existing item is assigned as each document is next edited; until then an untagged item is Q1.
- How the HAT compatibility levels L1–L3 and these quality levels combine for third-party HATs (a HAT can be L3 and Q1).
