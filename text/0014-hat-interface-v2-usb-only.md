# RFC 0014: HAT interface v2: HATs are USB only

- **Status:** Accepted (2026-10-25), with [hat-standard ADR 0006](https://github.com/opendap-project/hats/blob/main/standard/docs/adr/0006-hats-are-usb-only.md). Recorded in hat-standard, board and hat-camera; the other repository changes below are listed, not applied.
- **Date:** 2026-10-25
- **Affected repositories:** hat-standard (owner of the interface), board, hat-camera, hat-nas, hat-radio, hat-oscilloscope, hat-vna, hat-recorder, hat-powerbank, system, toolkit-verify.

## Summary

The HAT interface moves from v1.1 (68 pogo contacts in three levels: USB 3, a PCIe or CSI-2 extension, I2S, GPIO, SWD, I2C, an ID EEPROM, presence, interrupt) to **v2** (Draft): one USB port of **20 contacts** on a 2 × 10 field at 2.54 mm, carrying only `SSTX±`, `SSRX±`, `D±`, `VBUS` ×4, `GND` ×9 and `CC`. Every HAT capability goes through USB with standard device classes; power follows USB Power Delivery; identity is in USB descriptors, presence is attach and wake is USB remote wakeup. The specification is [interface v2](https://github.com/opendap-project/hats/blob/main/standard/docs/specs/interface-v2.md). v2 is a new major revision: v1 and v2 do not mate. Nothing was built against v1.

## Motivation

The maintainers decided on 2026-10-25 that HATs are USB only, and that charging and voltage management follow USB PD (hat-standard HCS-027 to HCS-030). The change removes every T1 part that existed only for the native HAT lines, and changes what each HAT repository builds (HCS-031: the HAT carries its own processing behind its USB port).

## Proposal

### 1. hat-standard (done in that repository)

Interface v2 (Draft), ADR 0006; ADR 0002, 0003 and 0005 superseded, ADR 0004 amended (field size; part and pitch stand); HCS-007, -015, -017, -022, -025, -026 withdrawn; HCS-027 to HCS-031 added.

### 2. board (T1): changes to make

#### BOM v1

| Line | Now | Change |
|---|---|---|
| 38 | HAT 3.3 V buck (TPS62903) | Remove: the interface has no 3.3 V rail |
| 41 | FUSB302BMPX ×1 (main USB-C) | ×2: a second Type-C port controller with USB PD for the HAT port (an address variant on I2C1 or the same part on another bus; the variants' addresses **UNVERIFIED**), mainline `fusb302` + `tcpm`; devicetree `usb-c-connector` with power-role dual, data-role host; dead-battery Rd so a source HAT can charge a flat T1 (**UNVERIFIED** on this part) |
| 44 | eFuse ×3 (#A HAT 5 V, #B HAT 3.3 V, #C USB-C) | ×2: #B removed. #A becomes the HAT `VBUS` source, enabled by the TCPM source path. Its minimum trip must cover the 1.5 A it offers: at 2.21 kΩ it is 1.344 A (1.524 A typical, ±10 %, 1 % resistor); **RILM 1.87 kΩ** gives 1.589 / 1.783 / 1.981 A (min / typ / max), inside the 4.5 A ceiling. #C has the same setting and the same question for the main USB-C. `VBUS` discharge on detach (#A output discharge **UNVERIFIED**; otherwise a discharge FET) |
| 45 | TPS2121 IN2 (powerbank `HAT_5V` in) | Stays as `VBUS` in from a source or dual-role HAT; its 2.95 A maximum trip is inside the 4.5 A ceiling. Decide which PD voltages the T1 accepts from a HAT: above 5 V, the OV2 threshold and the input path follow, as on the main USB-C. The OV2 interlock stays wired to the `VBUS` source enable |
| 65 | TVS0500 ×2 (HAT 5 V and 3.3 V) | ×1 on `VBUS`; a part rated for the highest voltage the T1 sources or accepts on the HAT port |
| 66 | DF40C-100DS receptacle, 1 per end | **DF40C-50DS** (with lock), 1 per end; 42 of 46 positions used (interface v2 §9.2) |
| 67 | P70-1000045R ×68 (4 × 17) | **×20** (2 × 10, 22.86 × 2.54 mm span, ≈ 18 N); Mill-Max 0965 second source unchanged |
| 68 | TPD4E05U06 ×4 (8 pairs) | **×1** (`SSTX±`, `SSRX±`) |
| 69 | TPD4E05U06 ×8 (22 low-speed + 8 reserved) | **×1** (`D±`, `CC`); `CC` protection rated for the highest `VBUS` on the port |
| 70 | 100 nF ×4 (USB 3 SSTX, PCIe TX) | ×2 (USB 3 SSTX only) |
| 94 | PCA9306 (HAT ID translator) | Remove |
| 95 | HAT audio translators (SN74AXC4T774, SN74AXC8T245) | Remove |
| 102 | HAT flat cable, DF40C-100DP headers | DF40C-50DP headers; power conductors for 3.0 A continuous and 4.5 A fault (15 + 15 contacts); USB pairs at 90 Ω differential; the 100 Ω extension pairs and the 50 Ω I2S lines go |
| 112 | HAT `I2S_MCLK` buffer (Open) | Remove (with the divider question it raised on the master-clock net) |
| 135 | SN74LVC1G57 ×2 (G3, G14) | ×1: G14 (PCIe reset gate) removed; G3 stays |
| 144 | TMUXHS4412 ×2 (extension switch) | Remove |

#### Pinout v1, HAT section

| Ball | Now | Change |
|---|---|---|
| AG37 (GPIO0_A5, PMUIO1) | `HAT_DET_N` | HAT Type-C port controller interrupt (wake-capable) |
| AJ38 (GPIO0_A6) | `HAT_INT_N` | Freed |
| A22, B22 (GPIO1_A0/A1) | `HAT_EXT_SEL`, `HAT_EXT_PD` | Freed |
| B24 (GPIO1_A5); A35, B34 (I2C4 M1) | `HAT_ID_EN`; `HAT_ID_SDA/SCL_1V8` | Freed (I2C4 may carry the second port controller) |
| 1A14 (GPIO1_A6); 1C15 (GPIO1_D4) | `HAT_MCLK_EN`; `HAT_AUD_OE_N` | Freed |
| B26, 1B13, A26 (GPIO1_B2/B0/B1) | `SOC_EXT_SB0`, `HAT_EXT_SB1/SB2` | Freed |
| 1B14 (GPIO1_B3) | `HAT_RST_N` | Freed |
| AP5, AR4, 1P3, 1P4 | `HAT_GPIO0`–`HAT_GPIO3` | Freed |
| AM2, AM1, 1R2, 1T1, AD1 | HAT I2S `DIN0`–`DIN3`, `DOUT` | Freed (the DAC I2S nets lose their HAT taps) |
| AR12, AP11 (I2C5 M1) | `HAT_I2C_SCL/SDA` | Freed (closes conflict C1 with the HDMI DDC balls) |
| CSI-2 RX (10 balls) | `HAT_CSI_*` | No consumer: unused-PHY handling (pinout open item O2) |
| PCIe REFCLK, TX, RX (6 balls) | `HAT_PCIE_*` | No HAT consumer: unused-PHY handling, unless the board gives PCIe another consumer |
| 1A19 (SARADC_VIN3) | eFuse #B IMON | Freed: can take eFuse #C IMON (closes C10) |
| Expander P01, P16 | `HAT3V3_EN`, `EFUSE_HAT3V3_FLT_N` | Freed; P02 `HAT5V_EN` becomes the `VBUS` source enable driven with the port controller |
| USB3 HOST1 (SS pairs, D±) | HAT USB 3 port | Unchanged |

**Requirements:** BRD-045 (PCIe straight to the HAT) no longer applies: withdraw or reword (board owner). BRD-044 and BRD-006 stand. BRD-042 has no HAT switch left to govern.

**Power tree §4.6–§4.7:** the HAT rails become one `VBUS` port under USB PD: source at 5 V up to 1.5 A (shared with the main USB-C as today), sink through TPS2121 IN2 at the voltages the board accepts; `HAT_3V3` and its budget go; the HAT fault ceiling becomes 4.5 A on `VBUS` (3.0 A continuous).

**PCB design constraints:** remove the extension rules (100 Ω `EXT` pairs, switch stubs, CSI-2 lane-to-clock matching through the switch, PCIe AC-cap placement before the switch); keep USB 90 Ω for SuperSpeed and `D±`; `CC` as a guarded slow line; `VBUS` and `GND` copper for 3.0 A continuous and 4.5 A fault.

**Feature blocks:** `blk.hat-5v` → the HAT `VBUS` source and sink, with the port controller (USB PD); `blk.hat-3v3` removed; `blk.hat-audio` removed; `blk.hat-signals` → USB 3, `D±` and `CC` on a DF40C-50, two ESD parts, no switches, no ID translator; gap G14 closes.

**HAT flat cable spec:** the DF40C-50 map of interface v2 §9.2, width and conductor sizing, coupon members (USB 3 Gen 1 and USB 2.0 only).

**Open on the board:** USB3 HOST1 remote wakeup from the T1's deep suspend (the HAT wake path now depends on it).

### 3. HAT repositories: requirement changes to make

Each HAT keeps its rows; `xxx-003` (mounts through the HAT Case Standard) now means interface v2. Proposed new rows (Draft):

| Repository | New row |
|---|---|
| every HAT | The HAT shall reach the device only through its USB port and meet the minimum conditions of hat-standard interface v2 (HCS-027 to HCS-031): standard USB classes, the identity capability, USB PD power rules, firmware recoverable over USB. |
| hat-camera | The HAT shall carry its own image processing and present video as UVC: an ISP for uncompressed video within a USB 3 Gen 1 link, and an encoder for modes beyond it. On-board AI (CAM-004) runs on the HAT. |
| hat-nas | The HAT shall present its SSD as USB mass storage (UAS) and its Ethernet ports as USB network functions, behind a USB 3 hub. |
| hat-radio | The HAT shall stream over High-Speed USB 2.0, as its base design does. |
| hat-oscilloscope | The HAT shall stream samples over High-Speed USB from its own MCU or FPGA. |
| hat-vna | The HAT shall use CDC-ACM, as its base design does. |
| hat-recorder | The HAT shall present audio as USB Audio Class 2 (asynchronous), with its own clock and its own monitoring output. |
| hat-powerbank | The HAT shall be a USB PD source or dual-role HAT, report its battery over PD, and keep the device as USB host. |

### 4. Other repositories

- **system / Blackbird OS:** HAT handling moves from EEPROM, overlays and native lines to USB: read the identity capability (BOS) and the signed record, resolve the manifest, bind class drivers; PD policy for the HAT port (offers, sink limits, role swaps, battery status) through TCPM; say clearly what a HAT needs and the device cannot supply (T1-045).
- **toolkit-verify:** the v2 conformance suite: `CC` and PD behaviour, `VBUS` limits and discharge, identity capability, USB signal quality through the field.

## Alternatives

- **Keep v1.1** (68 contacts): native lines and three levels; rejected by the decision.
- **A USB core plus a direct PCIe x1 extension** (52 contacts): keeps native NVMe and the SoC's PCIe in use; rejected by the decision (USB only).
- **A 2.0 A `VBUS` rating** (18 contacts): two fewer contacts, but below the 3.0 A that USB defines at 5 V and that PD contracts reach.
- **A 3 × 7 field** (21 contacts): shorter, but wider than a 2-row strip; no gain.
- **No `CC`:** VBUS would have to be on at the exposed field before any HAT is detected, and two power sources could meet; replaced by the PD decision.

## Open questions

- Signal-integrity coupon for USB 3 Gen 1 and USB 2.0 through the 2 × 10 field and the DF40C-50 FPC.
- Identity capability UUID and byte layout; whether common USB hubs and bridges accept custom BOS descriptors.
- Voltage rating of the pogo part above 5 V; which PD voltages the T1 accepts from a HAT. The HAT connector (pogo field, Mill-Max 0965) and the PD voltages (5 V only) were decided in [RFC 0015](0015-hat-connector-pogo-field-and-usb-c-port-hat.md).
- Remote wakeup from the T1's deep suspend.
