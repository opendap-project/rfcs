# RFC 0016: Outdoor HAT repository

- **Status:** Accepted (2026-10-25). The repository is created with its requirements; the changes in other repositories are listed, not applied.
- **Date:** 2026-10-25
- **Affected repositories:** workspace (`repos.tsv`), hat-outdoor (new), hat-standard, t1, system.

## Summary

A new reference HAT, the **Outdoor HAT**, extends the device for use beyond infrastructure: two-way satellite messaging with an emergency SOS, multi-band GNSS, LoRa mesh messaging and its own battery that powers the HAT and charges the device. It gets its own public repository, `hat-outdoor`, in `public/hats/outdoor`, scaffolded like the other HATs, with requirement prefix `OUT`. Cellular connectivity is a population option on a socket every unit carries. The HAT follows hat-standard interface v2: USB only, a dual-role USB PD HAT. Nothing here is built or measured.

## Motivation

- Every HAT is its own complete project, with hardware, case, firmware, software, a Blackbird OS package and its own tooling (the RAD-001 and PWR-001 pattern), so a new HAT needs its own repository.
- The T1 has no GNSS receiver, no satellite or LoRa radio and no room for another antenna next to its own (board ADR 0018). Functions that need those antennas go on a HAT.
- A satellite transmit peak exceeds what the T1 offers a HAT (5 V at up to 1.5 A, hat-standard interface v2 §5.2), so this HAT runs its radios from its own battery.
- LoRa mesh messaging has to keep relaying while the device sleeps, so it runs on the HAT's own microcontroller and battery.
- Adding a HAT to the reference list of interface v2 (§10) and naming a representative HAT for the T1's RF exposure (T1-076) touch other repositories; a change that spans repositories goes through an RFC.

## Proposal

### 1. hat-outdoor (new repository, applied)

- Repository `opendap-project/hat-outdoor`, folder `public/hats/outdoor`, with the layout and licences of the other HATs (REUSE; GPL-3.0-or-later by default, CERN-OHL-S-2.0 for `hardware/` and `case/`, MIT for `firmware/`, CERN-OHL-W-2.0 for `export/`, CC-BY-SA-4.0 for documentation) and the shared pre-commit checks.
- Requirements OUT-001 to OUT-015 in `docs/requirements/README.md`, all **Draft**: the HAT as a complete project; satellite messaging with SOS, multi-band GNSS (at least L1 and L5), LoRa mesh and its own battery (OUT-005); USB only within interface v2 (OUT-006); a dual-role USB PD HAT that sources the device within its input capability (OUT-007); an emergency SOS that works from the HAT alone (OUT-008); mesh, position logging and the satellite queue running while the device sleeps (OUT-009); cellular as a population option (OUT-010); barometer, magnetometer and flashlight (OUT-011); its own USB-C charging input (OUT-012); IP67 (OUT-013); a standard, user-replaceable cell (OUT-014); and transmitters that follow the device's certification rules for each certified variant (OUT-015).

### 2. workspace (applied)

- `repos.tsv`: `public/hats/outdoor` → `opendap-project/hat-outdoor`.

### 3. Other repositories (listed, not applied)

| Repository | Change |
|---|---|
| hat-standard | Interface v2 §10: add the Outdoor HAT row (a USB hub with the HAT's microcontroller as a composite device: identity capability, CDC-ACM for GNSS, satellite and mesh, HID sensors and HID power device; the cellular module where fitted; dual-role USB PD). |
| t1 | T1-076: whether this HAT is the representative HAT for the T1's RF-exposure assessment. |
| system | Packages for the HAT: ModemManager, gpsd, the Meshtastic host library, HID sensors, the PD battery view, satellite and mesh messaging, and offline maps fed by the HAT's position. |

## Alternatives

- **Outdoor functions on the T1 board.** Not chosen: each one needs an antenna the T1 body cannot host (board ADR 0018), or serves only this use.
- **LoRa mesh on the Radio HAT.** Not chosen for this use: the Radio HAT is a wideband SDR instrument, and only one HAT mounts at a time; a hiker needs the mesh together with GNSS, satellite and the battery. The Radio HAT keeps LoRa as a secondary function where fitted (RAD-016).
- **Running the radios from the T1.** Not chosen: a satellite transmit peak exceeds the 5 V, 1.5 A the T1 offers a HAT.

## Open questions

- Satellite module and its device approval route; the SOS response service.
- Cellular module, its current draw and its Linux USB support.
- Remote wakeup of the device from deep suspend (interface v2 §11 item 3).
- Whether the satellite path may draw from the T1's battery when the HAT's battery is empty.
