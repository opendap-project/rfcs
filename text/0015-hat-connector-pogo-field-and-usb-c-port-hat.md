# RFC 0015: HAT connector: pogo field, Mill-Max 0965, and the USB-C port HAT

- **Status:** Accepted (2026-10-25). Applied in hat-standard, board and t1; the case changes are listed, not applied.
- **Date:** 2026-10-25
- **Affected repositories:** hat-standard (owner of the interface), board, t1, case.

## Summary

With HATs USB only ([RFC 0014](0014-hat-interface-v2-usb-only.md), hat-standard interface v2), the physical HAT connector is decided: the **pogo field stays** (hat-standard ADR 0001), 20 contacts in a 2 × 10 grid at 2.54 mm, with the **Mill-Max 0965** as the primary pogo part and the **Harwin P70** as the alternate, each on its own hard-stop height (ADR 0004 amendment). A **USB-C port HAT** is added as a reference design: a HAT whose only function is a USB-C receptacle that exposes the device's HAT USB port, with its own USB PD. On the T1, the board items that waited for the connector are closed: the DF40C-50 flat-cable connectors, the pogo line, the flat cable, the ESD placement, a shared limiter that keeps the 5.1 V source within its rating in every fault case, and 5 V as the only voltage the T1 accepts from a HAT. Interface v2 keeps its contact map; nothing here is built or measured.

## Motivation

RFC 0014 left open whether the HAT mates through the pogo field on the device-side piece or through a USB-C receptacle, and the board kept BOM lines 66, 67 and 102, the flat-cable spec and the ESD placement waiting for it (hat-standard HCS-006, HCS-014, HCS-023; board BRD-013, BRD-049).

## Proposal

### 1. hat-standard (applied)

- **ADR 0001 confirmed:** the pogo field on the fixed device-side piece is the HAT connector of interface v2.
- **ADR 0004 amended:** Mill-Max 0965-0-15-20-80-14-11-0 primary (2 A, 1,000,000 cycles, working height 1.93–2.54 mm), Harwin P70-1000045R alternate (2.0 A, 10,000 cycles, 2.70–3.00 mm). The HAT pads are common; the device-side piece is built for one part, with its hard stops at **2.24 mm** (0965) or **2.85 mm** (P70) above the piece's land surface. 2 × 10 = 20 contacts confirmed (2 × 8 does not hold the v2 contact rules).
- **Interface v2 §9.1:** spring force **22.6 N** at mid stroke with the 0965 (17.6 N, 11.8–23.4 N, with the P70); retention preloaded for 23.4 N with margin; a **pad-plane tolerance rule**: 1.5 × the RSS of the pad-plane contributors stays within the populated part's half working window (0965 ±0.30 mm, P70 ±0.15 mm).
- **Interface v2 §5.2:** the T1 accepts **5 V only** from a HAT. **§9.2:** DF40C-50 genders; the ESD protection on the device-side piece at the field. **§10:** the USB-C port HAT.
- **Reference designs:** the USB-C port HAT row (§5 below).

### 2. board (applied: BOM v1.14, power tree v0.8, HAT flat cable spec v1)

| Item | Change |
|---|---|
| BOM line 66 | One Hirose DF40C-50DS-0.4V(51) receptacle per end (board and device-side piece); 46 usable positions, 42 used |
| BOM line 67 | 20 × Mill-Max 0965-0-15-20-80-14-11-0; Harwin P70-1000045R alternate (own piece land and stop height) |
| BOM line 102, flat cable spec | The interface v2 §9.2 map on DF40C-50DP-0.4V(51) headers; USB pairs 90 Ω (72.5 / 100 µm on L2, 89.9 Ω computed with `odapkit stackup`); `VBUS` ≥ 18.2 mm on L3 for the 4.5 A fault ceiling; the 100 Ω extension, 50 Ω I2S and 3.3 V classes removed |
| BOM lines 65, 68, 69 | TVS0500 on `VBUS` and the two TPD4E05U06 on the device-side piece, at the pogo field |
| BOM line 44, power tree §4.6 | A third TPS259470A (eFuse #D) between the 5.1 V buck and eFuses #A and #C, RILM 1.47 kΩ: trips at 2.021 / 2.268 / 2.520 A. With both ports at their maximum trip the buck carries at most 2.52 A, 84 % of its 3 A rating (was 3.96 A, 132 %); #D's minimum trip is above #A's and #C's maximum (1.981 A), so an overload on one port while the other is idle is cut by that port's eFuse |
| BOM line 45, power tree §4.7 | TPS2121 OV2 divider 464 kΩ / 100 kΩ: the HAT input turns off at 5.60–6.31 V. The T1 accepts 5 V only from a HAT |
| BRD-004 | Reworded: the SoC reaches every chip on the board directly and every HAT chip through the HAT's USB port |
| Pinout v1, PCB constraints, rules, feature blocks | Expander P16 takes eFuse #D's fault; PCB-P-04 at 2.52 A |

### 3. t1 (applied)

The electronics integration and verification spec (M-HAT, V2-H rows, CORE + HAT) and the faceplate interface follow interface v2 (USB only).

### 4. case (to make)

- Hard stops on the device-side piece at the populated part's height (0965: 2.24 mm above the land surface) and the pad-plane contributors of interface v2 §9.1 on the drawing.
- HAT retention preloaded for 23.4 N with margin.
- Flat-cable route, length (≤ 100 mm) and width (≥ 22.5 mm) for the board's HAT flat cable spec §12.

### 5. Reference design: USB-C port HAT

- **What it is:** a HAT whose only function is one USB-C receptacle on its outside. It exposes the device's HAT USB port to standard USB-C devices and chargers, is the reference HAT for makers, and is the path to prototype with any USB-C board.
- **Interface to the device:** interface v2 pads (all 20 contacts), stiffener under the pads.
- **Data:** a USB 3 hub on the HAT carries the OpenDAP HAT identity capability (interface v2 §7) and feeds one downstream USB-C receptacle: USB 3 Gen 1 and USB 2.0, reversible (`CC1`, `CC2`, SuperSpeed orientation mux). The device stays USB host, so the receptacle is a host port: it cannot connect the device to a computer.
- **Power:** a dual-role HAT (interface v2 §5.2) with its own USB PD controller on the receptacle. It sinks from the device and sources what is plugged in, within the device's offer (T1: 5 V, up to 1.5 A, less the HAT's own draw); with a charger plugged in, it becomes a USB PD source to the device at the voltages the device accepts (T1: 5 V). The receptacle's `VBUS` and `CC` are protected as an external USB-C port.
- **Firmware:** every programmable chip recoverable over USB (interface v2 §8).
- **Where:** hat-standard `reference-designs/usb-c-port/` (electronics and case). Status: proposed, not started.

## Alternatives

- **A USB-C receptacle in the device-side piece, with a USB-C plug on every HAT:** not adopted. The USB-C port is offered as the reference HAT instead, so flat HATs keep the pogo field.
- **Harwin P70 as the primary part (ADR 0004 as first written):** kept as the alternate, on its own hard-stop height.
- **A larger 5.1 V buck, or lower eFuse settings:** settings whose sum stays within the 3 A buck cannot also keep each port's minimum trip above its 1.5 A offer (two maximum trips of at least 1.87 A each); a shared limiter keeps the existing buck and both offers.

## Open questions

- Signal-integrity coupon (hat-standard ADR 0004) with the 0965 field, the device-side piece, the DF40C-50 and the FPC.
- HAT pad diameter and the case's pad-plane contributors (interface v2 §11).
- PD voltages above 5 V from a HAT: a later board revision (TVS, `CC` protection, OV2) if wanted.
