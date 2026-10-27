# RFC 0013: Faceplate interface v1

- **Status:** Accepted (2026-10-24).
- **Completes:** [RFC 0001](0001-faceplate-interface-v0-pinout.md)
- **Date:** 2026-10-24 (numbered 0013: 0012 was already taken by the composer RFC)
- **Affected repositories:** t1 (owner of the interface), board, system, faceplate-touch, faceplate-classic, faceplate-cassette, toolkit-verify.

## Summary

The faceplate interface moves from v0 (signal groups, RFC 0001) to v1 (Draft): contact positions, electrical classes and supply limits, the HID-over-I2C control and identity protocol, capability declaration, versioning, a mechanical parameter table and conformance tests. The specification is [faceplate-interface.md](https://github.com/opendap-project/t1/blob/main/docs/specs/faceplate-interface.md), with t1 ADR 0004 (identity and HID transport) and ADR 0005 (supply classes and protection).

## Motivation

T1-003, T1-018 to T1-024 and T1-036 require interchangeable faceplates on one interface. v0 left the contact positions, the I2C address and descriptor register, the report layout, identification, power behaviour in suspend, supply protection, dimensions and anchoring undefined, so no faceplate or board schematic could be drawn against it.

## Proposal

- **Interface (t1):** the v1 specification. Positions are those of board pinout v1 §8; 5 of 51 contacts stay reserved (9.8 %). No compatibility levels: one pinout, and a display is a rule of each faceplate.
- **Identity and control (t1 ADR 0004):** every faceplate carries an HID-over-I2C identity controller at 0x2C, descriptor register 0x0001, level-triggered interrupt; vendor ID 0x1209 through pid.codes (test IDs allowed up to RFC 0010 Q1); vendor usage page 0xFF00; a mandatory identity report carries the version that mainline `i2c-hid` hides.
- **Supplies (t1 ADR 0005):** the body current-limits every faceplate rail (FP_3V3 ≤ 0.4 A, FP_1V8 and MIC_1V8 ≤ 0.2 A trip); the backlight line never exceeds 30 V in any state; faceplate rails are off in suspend by default.
- **Board:** current-limited faceplate switches, backlight over-voltage limit, identity node in the devicetree, I2C pull-ups for 400 kHz (board BOM v1.11).
- **System:** the H3 faceplate description maps the identity report to the faceplate's device description (system hardware-support spec §2–§3); description sections for microphones and per-rail power to add.
- **Faceplate repositories:** each faceplate adds its identity controller and declares its capabilities.
- **toolkit-verify:** the 32 conformance rows of the specification become the faceplate suite.

## Alternatives

- **EEPROM identification (T1-020's fallback):** kept out; the HID descriptor already identifies the faceplate and a controller is needed for the controls anyway.
- **Reducing the pinout:** reviewed and rejected; the 14 contacts no faceplate uses yet carry the display and touch standards T1-036 and BRD-048 require, and the next connector size cannot hold the 46 used contacts.

## Open questions

- The pid.codes allocation is requested by the maintainers; their rules mention USB devices, so the fallback (a USB-IF vendor ID or a chip maker's programme) may be needed.
- The FH35C DC voltage rating (**UNVERIFIED**) and every mechanical value marked "input from the case".
